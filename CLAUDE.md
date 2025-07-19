# Home-Ops Kubernetes Cluster - Claude Context

## Project Overview
This is a **production-ready Kubernetes cluster template** for home operations based on onedr0p/cluster-template. It provides a complete GitOps-based infrastructure-as-code solution using Talos Linux, Kubernetes, and Flux for deploying and managing a secure, automated home lab cluster.

## Quick Reference

### Cluster Information
- **Cluster Network**: 10.0.0.0/24
- **Domain**: x00.sh (Cloudflare managed)
- **Kubernetes Version**: v1.33.3
- **Nodes**: 3 controller nodes (kube-control-01/02/03)
- **Node IPs**: 10.0.0.156, 10.0.0.157, 10.0.0.158

### Virtual IPs
- **Kube API Server**: 10.0.0.50
- **Internal Gateway**: 10.0.0.52  
- **External Gateway**: 10.0.0.51
- **DNS Gateway**: 10.0.0.53

### Key Commands
```bash
# Initialize from samples
task init

# Render templates and validate
task configure

# Bootstrap Talos cluster
task bootstrap:talos

# Install core apps
task bootstrap:apps

# Force Flux sync
task reconcile

# Check cluster status
kubectl get nodes
flux get kustomizations
```

## Architecture Stack

### Core Technologies
- **Talos Linux**: Immutable, minimal Kubernetes OS
- **Kubernetes**: Container orchestration (3-node HA cluster)
- **Flux**: GitOps continuous delivery
- **Cilium**: eBPF-based CNI with Gateway API
- **SOPS**: Secret management with age encryption
- **Cloudflare**: DNS, tunneling, and external access

### Infrastructure Tools
- **Makejinja**: Template rendering with custom delimiters `#{...}#` and `#%...%#`
- **Talhelper**: Talos configuration management
- **Helmfile**: Declarative Helm chart deployment
- **Task**: Automation and workflow management
- **Mise**: Development environment tool management

## Project Structure

```
home-ops/
├── cluster.yaml              # Main cluster configuration
├── nodes.yaml                # Node definitions (IPs, MACs, disks)
├── makejinja.toml            # Template rendering config
├── Taskfile.yaml             # Main automation tasks
├── kubernetes/               # All K8s manifests
│   ├── apps/                 # Application deployments
│   │   ├── cert-manager/     # TLS certificate management
│   │   ├── echo/             # Sample application
│   │   ├── flux-system/      # GitOps operator
│   │   ├── kube-system/      # Core K8s components
│   │   └── network/          # Networking stack
│   ├── components/           # Shared components
│   └── flux/                 # Flux configuration
├── talos/                    # Talos Linux configs
│   ├── talconfig.yaml        # Main Talos config
│   └── patches/              # Node-specific patches
├── templates/                # Jinja2 templates
├── bootstrap/                # Initial setup configs
├── scripts/                  # Automation scripts
└── .taskfiles/              # Modular task definitions
```

## Configuration Management

### Template System
- **Engine**: Makejinja with Jinja2 templates
- **Data Sources**: cluster.yaml, nodes.yaml
- **Custom Delimiters**: `#{variable}#` and `#%block%#`
- **Output**: Rendered configs in respective directories

### Secret Management
- **Encryption**: SOPS with age encryption
- **Key Storage**: `.age.key` (excluded from Git)
- **Scope**: Separate patterns for Talos vs Kubernetes secrets
- **Convention**: `.sops.yaml` defines encryption rules

### GitOps Workflow
- **Source of Truth**: Git repository
- **Sync**: Flux monitors for changes
- **Structure**: Kustomization hierarchy by namespace
- **Automation**: GitHub webhooks for immediate sync

## Networking Architecture

### CNI: Cilium
- **Mode**: Native routing (no encapsulation)
- **Features**: BGP, Gateway API, L2 announcements
- **Load Balancer**: DSR mode for efficiency
- **Security**: Network policies, eBPF enforcement

### DNS Strategy
- **Internal**: k8s_gateway for cluster services
- **External**: Cloudflare for public domains
- **Resolution**: CoreDNS with custom forwarding
- **Integration**: external-dns for automatic record management

### External Access
- **Method**: Cloudflare Tunnels (no port forwarding)
- **Security**: Zero-trust access model
- **TLS**: cert-manager with Let's Encrypt
- **Ingress**: Gateway API with Cilium

## Deployment Process

### 5-Stage Bootstrap
1. **Machine Prep**: Talos ISO creation and node imaging
2. **Workstation**: Tool installation via Mise
3. **Cloudflare**: API token and tunnel configuration
4. **Configuration**: Template rendering and validation
5. **Bootstrap**: Talos → Kubernetes → Flux deployment

### Critical Files to Configure
- `cluster.yaml`: Network, domain, tokens
- `nodes.yaml`: Node specs (IP, MAC, disk, schematic)
- `.age.key`: SOPS encryption key
- `bootstrap/flux/github-webhook-token.sops.yaml`: GitHub integration

## Included Applications

### Core Infrastructure
- **cilium**: CNI with advanced networking
- **cert-manager**: Automatic TLS certificates
- **external-dns**: DNS record automation
- **reloader**: Config change detection
- **spegel**: Registry mirror for efficiency

### Monitoring & Observability
- **metrics-server**: Resource metrics collection
- **Prometheus**: (via component configs)
- **Grafana**: (via component configs)

### Development & Testing
- **echo**: Sample application for validation
- **flux-system**: GitOps operator and dashboards

## Security Considerations

### System Security
- **Immutable OS**: Talos reduces attack surface
- **Encrypted Secrets**: All sensitive data via SOPS
- **Non-root**: Containers run as non-privileged users
- **Network Policies**: Micro-segmentation via Cilium

### Access Control
- **RBAC**: Kubernetes role-based access
- **Service Mesh**: Optional Istio integration
- **Zero Trust**: Cloudflare Access integration
- **Audit**: Kubernetes audit logging enabled

## Development Workflow

### Git Workflow
- **Conventional Commits**: Use semantic commit messages (feat:, fix:, docs:, etc.)
- **Feature Branches**: Create feature branches for all changes
- **Pull Requests**: All changes must go through PR review process
- **Branch Protection**: Main branch requires PR approval before merge

### Tools Management
```bash
# Install/update tools
mise install

# Verify environment
mise list
```

### Template Development
```bash
# Render templates
task configure

# Validate schemas
task validate

# Check differences
task diff
```

### Testing
```bash
# Local Flux validation
flux-local test

# Dry-run deployments
task bootstrap:test
```

## Troubleshooting

### Common Issues
- **SOPS Errors**: Check age key permissions and location
- **Template Failures**: Validate Jinja2 syntax and data
- **Flux Sync**: Check GitHub webhook token and connectivity
- **Cilium**: Verify BGP configuration and network policies

### Useful Debug Commands
```bash
# Talos logs
talosctl logs -n <node-ip>

# Flux status
flux get all --all-namespaces

# Cilium connectivity
cilium connectivity test

# Pod networking
kubectl exec -it <pod> -- nslookup kubernetes.default
```

### Log Locations
- **Talos**: `talosctl logs`
- **Kubernetes**: `kubectl logs`
- **Flux**: `flux logs`
- **Applications**: Namespace-specific logs

## Maintenance

### Updates
- **Automated**: Renovate handles dependency updates
- **Manual**: Review and merge PR updates
- **Testing**: flux-local validates before deployment

### Backup Strategy
- **GitOps**: All config in Git (recoverable)
- **Secrets**: Age key backup essential
- **Persistent Data**: Application-specific backup strategies

### Monitoring
- **Cluster Health**: Kubernetes dashboard
- **GitOps Status**: Flux dashboard
- **Application Metrics**: Prometheus/Grafana stack

## MCP Server Integration

### Available Servers
- **memory**: Persistent session memory (`.memory.json`)
- **context7**: External context and AI assistance
- **kubernetes**: Direct cluster interaction and management
- **ide**: VS Code diagnostics and code execution

### Usage Patterns
- Use **kubernetes MCP** for cluster operations
- Use **memory MCP** to persist important context
- Use **ide MCP** for development and debugging
- Use **context7** for enhanced AI capabilities

## Best Practices

### File Management
- **Always** use template system for configs
- **Never** edit rendered files directly
- **Commit** template changes, not rendered output
- **Validate** before applying changes

### Security
- **Rotate** age keys periodically
- **Audit** secret access patterns
- **Monitor** for suspicious activities
- **Update** dependencies regularly

### Operations
- **Test** changes in non-production first
- **Backup** before major updates
- **Document** custom modifications
- **Monitor** deployment health

---

*This file provides essential context for Claude to understand and work with the home-ops Kubernetes cluster. Keep it updated as the cluster evolves.*