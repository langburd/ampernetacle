# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Terraform config that deploys a kubeadm-bootstrapped Kubernetes cluster on Oracle Cloud Infrastructure (OCI) using ARM instances (Ampere Altra). Designed for learning/dev — not production.

## Apply Workflow

Always follow this sequence:
```
terraform validate
terraform plan
terraform apply
```

## Backend

- State stored in OCI Object Storage (S3-compatible) in `il-jerusalem-1`
- Requires `~/.oci/config` with profile `ddyyconsulting`
- Session tokens expire in 1 hour — refresh with: `oci session refresh --profile ddyyconsulting`

## Non-Obvious Gotchas

- **State upload failures (chunked encoding)**: OCI Object Storage doesn't support AWS chunked transfer encoding. If Terraform fails with `api error NotImplemented: AWS chunked encoding not supported`, it writes `errored.tfstate` locally. Push it manually:
  ```bash
  oci os object put \
    --namespace axbasucxrqax \
    --bucket-name ddyyconsulting-terraform-states \
    --name ampernetacle-cluster-in-il/terraform.tfstate \
    --file errored.tfstate \
    --profile ddyyconsulting \
    --force
  ```
- **Capacity errors**: If OCI returns "Out of host capacity", change `availability_domain` variable (0, 1, or 2)
- **Region mismatch**: `~/.oci/config` region must match the OCI account's home region (visible in OCI console URL)
- **Cloud-init timing**: `remote-exec` blocks on `cloud-init status --wait` — applies can take 10+ minutes; this is expected
- **Kubeconfig**: After apply, run `export KUBECONFIG=$PWD/kubeconfig` before using kubectl
- **No LoadBalancer support**: No OCI Cloud Controller Manager — use NodePort instead
- **Weave CNI**: Version pinned to v2.8.1 in `cloudinit.tf`

## Resource Naming Convention

All resources use single underscore `_` as the local name (e.g., `resource "oci_core_vcn" "_"`). This is intentional — keep it consistent.

## Key Variables

- `k8s_version` — Kubernetes version to install (e.g., `"1.33"`)
- `how_many_nodes` — total node count (1 control plane + N-1 workers)
- `shape` — instance shape; `VM.Standard.A1.Flex` is ARM and qualifies for free tier
