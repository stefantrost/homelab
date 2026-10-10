homelab/
├── README.md
├── docs/
│   ├── network.md          # IP plan: nodes, LB pool, pod/service CIDRs, DNS names
│   └── runbooks/           # upgrade, etcd restore, adding a node
├── infrastructure/         # outside the cluster, applied by hand
│   ├── proxmox/            # template build steps, VM specs
│   ├── node-prep/          # containerd config, sysctl file, install script
│   └── kubeadm/
│       └── kubeadm.yaml
├── cluster/                # inside the cluster, managed by Argo CD later
│   ├── argocd/             # Argo CD itself + the root "app of apps"
│   ├── platform/
│   │   ├── cilium/         # Helm values
│   │   ├── cert-manager/
│   │   ├── gateway/
│   │   └── monitoring/
│   └── apps/
│       └── <one folder per app>/
└── .gitignore
