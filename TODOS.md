# Additionals / TODOs

## Partial occupancies

MACE assumes fully occupied sites: each atom has a single element type, and
`node_attrs` is a one-hot vector over atomic numbers. Many experimental crystal
structures (e.g. from the ICSD or MP) contain fractional/disordered occupancies
where a site is shared between two or more species.

Things to consider:
- Represent partially occupied sites as a soft/mixed node attribute vector
  (weighted sum of one-hot vectors) rather than a hard one-hot.
- The radial embedding already handles continuous features; the main change
  is in `LinearNodeEmbeddingBlock` and anywhere `node_attrs` is used as a
  species index (e.g. `skip_tp`, symmetric contraction weights).
- May require changes to how `AtomicNumberTable` maps species → indices.
- Decide whether to handle at the data level (augmentation / random sampling
  of a single species per site) or at the model level (true soft embeddings).

## Per-head output normalisation

`ScalarPropertyMACE` currently outputs raw MLP values. Different datasets
(PBE, HSE, G0W0, GW100) have different mean and variance of bandgap, so
training is unbalanced without normalisation.

Suggested approach:
- Compute per-head mean and std from the training split at data-loading time
  (analogous to how `compute_mean_std_atomic_inter_energy` works for energy).
- Store as non-trainable buffers in `ScalarPropertyMACE` (one scalar per head).
- Apply as an affine transform on the output: `y = std * MLP(x) + mean`,
  so the MLP learns residuals around the per-head mean in units of std.
- This is cleaner than repurposing `ScaleShiftMACE`'s energy scale/shift,
  which is per-element and coupled to per-atom energy decomposition.
