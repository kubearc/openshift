# DO280 Chapter 1 — Student Practice Set
### Declarative Resource Management & Kustomize (independent practice)

Work through these on your own, in order — each one builds on the app you create in Task A. Try each task **before** opening the hint or solution. If you get stuck for more than a few minutes, open the hint first; save the full solution for after you've attempted it.

**Setup (do this once):**
```
oc new-project ch1-homework
```
Use this project for every task below. If you want a clean slate at any point: `oc delete project ch1-homework && oc new-project ch1-homework`.

Everything here uses a small app called **`shop`** — the same one from lecture (a Deployment, a ConfigMap, and a Service). You'll build it out declaratively from scratch and then bring it under Kustomize.

---

## Task A — From Imperative to a Manifest

**Goal:** Practice generating a manifest instead of writing one from memory.

Create a Deployment named `shop` using image `registry.access.redhat.com/ubi9/httpd-24`, but instead of running it directly, generate it as a YAML file you can review and clean up first. Then apply it for real and confirm the Pod reaches `Running`.

<details>
<summary>Hint</summary>

You need a flag that renders the object without contacting the cluster, and a flag that stores the configuration `oc apply` will need later. Both go on `oc create deployment`.
</details>

<details>
<summary>Full solution</summary>

```
oc create deployment shop -o yaml \
  --image registry.access.redhat.com/ubi9/httpd-24 \
  --save-config --dry-run=client > shop-deployment.yaml

# open shop-deployment.yaml, delete creationTimestamp: null, status: {}, strategy: {}

oc apply -f shop-deployment.yaml
oc get pods -w
```
**Self-check:** `oc get deployment shop` shows `1/1`. If it doesn't reach Running, run `oc describe pod` on the failing pod and check the Events section before asking for help.
</details>

---

## Task B — Prove `oc apply` Removes Dropped Fields

**Goal:** See the three-way merge behavior from lecture with your own hands, not just on a slide.

1. Add an environment variable `THEME=light` to the `shop` Deployment's container and apply it.
2. Confirm it's there with `oc set env deployment/shop --list`.
3. Now delete that environment variable from your YAML file (not set it to empty — remove the lines) and re-apply.
4. Check again. Is `THEME` still there?

<details>
<summary>Hint</summary>

The env var goes under `spec.template.spec.containers[0].env`, as a list of `{name, value}` pairs. This only behaves the way lecture described because your original file was created with `--save-config` — if you skipped Task A's flags, this experiment won't show the effect you're expecting.
</details>

<details>
<summary>Full solution</summary>

```
# add to shop-deployment.yaml under containers[0]:
#   env:
#     - name: THEME
#       value: light

oc apply -f shop-deployment.yaml
oc set env deployment/shop --list        # THEME=light

# remove the env: block entirely, then:
oc apply -f shop-deployment.yaml
oc set env deployment/shop --list        # THEME is gone
```
**Self-check:** Can you explain, out loud or in writing, *why* `THEME` disappeared? If your answer doesn't mention `last-applied-configuration`, go back and reread the three-way merge section of the trainer notes before moving on.
</details>

---

## Task C — Reproduce the Stale-Pod Problem Yourself

**Goal:** Trigger the exact ConfigMap staleness issue from lecture, then fix it two different ways.

1. Create a ConfigMap `shop-config` with a key `PROMO=none`.
2. Wire it into the `shop` Deployment as an environment variable.
3. Confirm the running Pod sees `PROMO=none`.
4. Update the ConfigMap to `PROMO=summer-sale` — do **not** touch the Deployment.
5. Check the Pod's environment again. What do you see?
6. Fix it using `oc rollout restart`.
7. Repeat steps 4–5 one more time, but this time fix it by deleting the Pod directly instead. What's different about how the app behaves during the fix, if you imagine this Deployment had 3 replicas instead of 1?

<details>
<summary>Hint</summary>

`oc exec deploy/shop -- printenv PROMO` is the fastest way to check what a running Pod actually has, without needing to `oc rsh` in.
</details>

<details>
<summary>Full solution</summary>

```
oc create configmap shop-config --from-literal=PROMO=none
oc set env deployment/shop --from=configmap/shop-config
oc exec deploy/shop -- printenv PROMO          # none

oc create configmap shop-config --from-literal=PROMO=summer-sale \
  --dry-run=client -o yaml | oc apply -f -
oc exec deploy/shop -- printenv PROMO          # still "none" — stale!

oc rollout restart deployment/shop
oc exec deploy/shop -- printenv PROMO          # now "summer-sale"
```
**Self-check (question 7):** with multiple replicas, `oc rollout restart` replaces Pods gradually — the app stays available throughout. Deleting a Pod directly on a multi-replica Deployment still works, but you lose the controlled, one-at-a-time rollout behavior. Write one sentence explaining which one you'd use in production and why.
</details>

---

## Task D — Targeted Changes with `oc patch`

**Goal:** Make a one-off change without hand-editing and reapplying the whole manifest.

Using `oc patch`, change the `shop` Deployment's replica count to `2` — without opening `shop-deployment.yaml` at all. Then check whether your YAML file and the live cluster now agree with each other.

<details>
<summary>Hint</summary>

`oc patch deployment shop -p '<JSON here>'` — the JSON needs to reach `spec.replicas`.
</details>

<details>
<summary>Full solution</summary>

```
oc patch deployment shop -p '{"spec":{"replicas":2}}'
oc get deployment shop
```
**Self-check:** Your YAML file still says `replicas: 1` — only the live cluster changed. If you ran `oc apply -f shop-deployment.yaml` again right now, what would happen to your replica count? Try it and see if your prediction was right.
</details>

---

## Task E — Build a Kustomize Base for `shop`

**Goal:** Turn what you've built so far into a reusable Kustomize base.

Create `shop-app/base/` with `deployment.yaml`, `service.yaml` (a ClusterIP Service exposing port 8080), and a `kustomization.yaml` listing both. Render it with `oc kustomize` before applying anything.

<details>
<summary>Hint</summary>

`kustomization.yaml` needs a `resources:` list with the filenames of the other two files — nothing more for a minimal base.
</details>

<details>
<summary>Full solution</summary>

```
mkdir -p shop-app/base
# copy your cleaned-up shop-deployment.yaml into shop-app/base/deployment.yaml
# write a matching service.yaml (ClusterIP, port 8080 -> containerPort)

cat > shop-app/base/kustomization.yaml <<'EOF'
resources:
  - deployment.yaml
  - service.yaml
EOF

oc kustomize shop-app/base
```
**Self-check:** The rendered output should contain both a `Deployment` and a `Service`, with no errors. Don't apply yet — Task F builds directly on this base.
</details>

---

## Task F — Staging & Production Overlays

**Goal:** One base, two environments, different replica counts and resource limits — without editing the base.

1. Create `shop-app/overlays/staging/` — `namePrefix: staging-`, 1 replica (the base default is fine).
2. Create `shop-app/overlays/production/` — `namePrefix: prod-`, patched to 3 replicas, with a CPU request of `200m` added to the container.
3. Render **both** overlays and compare the output side by side before applying either.
4. Apply `production` only, and confirm 3 Pods come up.

<details>
<summary>Hint</summary>

Two separate patch operations are cleanest here: one `op: replace` on `/spec/replicas`, and one `op: add` on the container's `resources` field. They can live in the same `patches:` entry or as two separate entries — try both and see which you find more readable.
</details>

<details>
<summary>Full solution</summary>

```
mkdir -p shop-app/overlays/staging shop-app/overlays/production

cat > shop-app/overlays/staging/kustomization.yaml <<'EOF'
namePrefix: staging-
resources:
  - ../../base
EOF

cat > shop-app/overlays/production/kustomization.yaml <<'EOF'
namePrefix: prod-
resources:
  - ../../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
    target:
      kind: Deployment
      name: shop
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: 200m
    target:
      kind: Deployment
      name: shop
EOF

oc kustomize shop-app/overlays/staging
oc kustomize shop-app/overlays/production

oc apply -k shop-app/overlays/production
oc get pods -l app=shop
```
**Self-check:** If `oc apply -k` fails or the replica count is wrong, the fix is almost never in the base — check your patch's `target.kind` and `target.name` match the base resource exactly.
</details>

---

## Task G — Generators: Automatic Rollouts

**Goal:** Replace the manual `oc rollout restart` from Task C with Kustomize's automatic mechanism.

1. Add a `configMapGenerator` to the `staging` overlay that creates `shop-config` with `PROMO=none`.
2. Wire it into the base Deployment as an environment source.
3. Apply the `staging` overlay, note the running Pod names.
4. Change `PROMO` to `winter-sale` in the overlay and reapply — **without running `oc rollout restart` yourself.**
5. Confirm new Pods appeared and have the new value.

<details>
<summary>Hint</summary>

Check the *actual* name Kustomize gives the generated ConfigMap with `oc get configmap` — it won't be `shop-config`. That renamed reference is what triggers the rollout; if you hardcoded `shop-config` as the reference name in your Deployment instead of letting Kustomize rewrite it, the rollout won't happen.
</details>

<details>
<summary>Full solution</summary>

```
cat >> shop-app/overlays/staging/kustomization.yaml <<'EOF'
configMapGenerator:
  - name: shop-config
    literals:
      - PROMO=none
EOF

# in base/deployment.yaml, add under the container spec:
#   envFrom:
#     - configMapRef:
#         name: shop-config

oc apply -k shop-app/overlays/staging
oc get pods -l app=shop
oc get configmap                      # note the hashed name

# edit PROMO=winter-sale in the overlay's kustomization.yaml, then:
oc apply -k shop-app/overlays/staging
oc get pods -l app=shop -w            # new pods roll out automatically
```
**Self-check:** Explain in one or two sentences why this achieves the same outcome as Task C's `oc rollout restart`, but without you having to remember to run it.
</details>

---

## Task H — Capstone (unguided)

No hints on this one — this is your check that everything above actually stuck.

> Starting from a fresh project, build a Kustomize base for any small app of your choosing (it doesn't have to be `shop`). Create two overlays — one with 1 replica and a generated ConfigMap-driven environment variable, one with 3 replicas and a resource request/limit patch. Render both, verify they differ correctly, then apply only the 3-replica one and confirm it's healthy.

<details>
<summary>Self-check rubric</summary>

- [ ] Base has zero environment-specific values
- [ ] Both overlays reference the base via `resources: [../../base]`
- [ ] Replica count and resource limits differ via a patch, not by editing the base
- [ ] You ran `oc kustomize` on both overlays before applying either
- [ ] The generated ConfigMap shows a hashed name in `oc get configmap`
- [ ] `oc apply -k` on the 3-replica overlay succeeds and all Pods reach `Running`

If every box is checked without peeking back at Tasks E–G, you've got this chapter down.
</details>

---

## When You're Done
```
oc delete project ch1-homework
```
