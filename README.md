# Flux Infra

I used to have two repositories for my flux configuration, but that got tiresome. Now 
they are combined. This allows me to re-use shared configuration, but maintain separate 
deployments.

## Layout

Shared configuration lives in `base/`, and we make use of flux `postBuild.Substitutions` 
to drop in cluseter specific values. Cluster information is stored under `clusters/${cluster}`. 


The main flux kustomizations are in the root directory, these include:

- _cluster-overrides: Custom config maps, often used as valuesFrom in HelmReleases
- applications.yaml: Points to the cluster specific deployments
- configuration.yaml: Points to CRDs implemented for specific clusters
- flux-sync.yaml: Points to the base flux setup which sets up the flux operator and flux instance
- infrastructure.yaml: Points to the base infrastructure setup.

### Base Directory
_./base/_

Contains shared infrastructure concepts. And some non-shared but might-be-shared components.

- _components: contains re-usable components that are not inherently applicable to all clusters
  - csi: CNI options. So far just local and NFS
  - metallb: Provides bare metal load balancing for bare metal clusters
- cert-manager: Cloud native certificate provisioning
- cilium: CNI. Non-optional, and opinionated.
- external-secrets: Operator for connecting to secrets stores.
- kube-state-metrics: Collects metrics on installed cluster objects.
- metrics-server: Used for VPA/HPA etc.
- node-exporter: Linux metrics
- tetragon: siem/security monitroing
- traefik: Cluster ingress/proxy

### Cluster Directory
_./clusters/${cluster}_

This contains cluster specific configuration. Sets up the base kustomizations and config 
maps that are used for building the cluster. It's the root that the FluxInstance points to. 

- _cluster-overrides: config maps that container cluster specific overrides
- applications: cluster specific deployments
- configuration: Implemntation of CRDs deployed by infrastructure

## Contributing

The kubeconform lint job reads `.kubeconformignore` to configure ignored paths.
Add one Go regular expression per line (not a glob); blank lines and comment lines
starting with `#` are ignored. Patterns are passed as `-ignore-filename-pattern`
arguments, so no workflow changes are needed when adding an exclusion.

This repository make use of [pre-commit](https://pre-commit.com/) to automatically run git hooks 
that makes sure that actions/tests will pass. Make sure to enroll before pushing.
