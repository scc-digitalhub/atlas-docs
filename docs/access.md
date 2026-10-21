# Access ATLAS tenants

ATLAS is a shared infrastructure, where every _tenant_ has a dedicated, isolated namespace.

The endpoints of the tenant platform components are available through the `https://*.<tenant>.atlas.fbk.eu` domain, for example:

- Main UI: `https://core.<tenant>.atlas.fbk.eu`
- Interactive workspace manager: `https://coder.<tenant>.atlas.fbk.eu`

The name of the tenant corresponds to the research unit name.

When accessing the components of the platform (either through the WEB interfaces, CLI or SDK), the users are authenticated using the FBK login credentials via SSO.

The access to the datalake is based on personal session token credentials emitted by the management layer. The database storage is not available outside the cluster. The interactive workspaces are also personal and are connected to the management layer and storages automatically.

!!! warning

    Please note that the cluster is available only within the FBK network.
    
    To access the cluster outside of this network, it is necessary to access it via a VPN connection. See a [dedicated FBK HowTo](https://howto.fbk.eu/documenti/zero-trust-vpn/) section on this topic.
