# Ad-hoc limited-trust mesh federation

Principles:

- use **standard Istio mechanisms** such as gateways, virtual services, destination rules, RBAC.
- use **standard Istio installations**.
- **ad hoc mesh federation** at any time. The owners of the meshes can install Istio and operate it independently,
  and decide to _federate_ the meshes at some later point in time.
- **private gateways for cross-cluster communication**, with dedicated certificates and private keys.
- **limited trust**: Only the gateways trust each other, there is no trust between sidecars from different meshes.
- use **standard Kubernetes DNS**, no need for special DNS plugins.

Examples:
* [HTTP](http)
* [TLS](tls)
* [TCP](tcp)
