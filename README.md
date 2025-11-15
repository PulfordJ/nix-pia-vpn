# Private Internet Access wireguard client on networkd for NixOS

NixOS configuration for connecting to PIA VPN via wireguard. Supports port forwarding. Supports
custom networkd `netdev` and `network` configuration. By default it will select the server with
the lowest ping, but it can be configured to connect to a chosen server (this is faster).

Based on [tadfisher's pia-vpn.nix](https://github.com/tadfisher/flake/blob/f6f9c5a/nixos/modules/pia-vpn.nix). Thanks!

## Usage

```
# flake.nix
{
  inputs = {
    nix-pia-vpn.url = "github:rcambrj/nix-pia-vpn";
    nix-pia-vpn.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = inputs@{ ... }: {
    nixosConfigurations = {
      my-host = inputs.nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          inputs.nia-pia-vpn.nixosModules.default
          {
            services.pia-vpn = {
              enable = true;
              certificateFile = ./ca.rsa.4096.crt;
              environmentFile = # use sops-nix or agenix
            };
          }
        ];
      };
    };
  };
}
```

### [optional] Set VPN as default route

```
# configuration.nix
  services.pia-vpn.networkConfig = ''
    [Match]
    Name = ''${interface}

    [Network]
    Description = WireGuard PIA network interface
    Address = ''${peerip}/32

    [RoutingPolicyRule]
    To = ''${wg_ip}/32
    Priority = 1000

    # if port forwarding is required, make an exception for that service
    # as it's not accessible from inside the VPN
    [RoutingPolicyRule]
    To = ''${meta_ip}/32
    Priority = 1000

    [RoutingPolicyRule]
    To = 0.0.0.0/0
    Priority = 2000
    Table = 42

    [Route]
    Destination = 0.0.0.0/0
    Table = 42
  '';
```


```
# configuration.nix
  services.pia-vpn.portForward = {
    enable = true;
    script = ''
      export $(cat transmission-rpc.env | xargs)
      ${pkgs.transmission_4}/bin/transmission-remote --authenv --port $port || true
    '';
  };
```

### [optional] Bring up services when VPN is up (and tear them down when it's not)

```
# configuration.nix
  systemd.services.my-service-which-depends-on-vpn = {
    after = [ "pia-vpn.service" ];
    bindsTo = [ "pia-vpn.service" ];
  };
```

### [optional] Network Namespace Isolation

The `namespace` option creates the WireGuard interface in an isolated network namespace instead of the main system namespace. This prevents the PIA service from disrupting other network services (like Tailscale) during startup, since the VPN interface is created in complete isolation from the host's networking.

Additionally, services bound to this namespace can **only** access the internet through the VPN, providing leak-proof isolation.

```
# configuration.nix
  services.pia-vpn = {
    enable = true;
    certificateFile = ./ca.rsa.4096.crt;
    environmentFile = ./pia.env;
    namespace = "wg";  # Create in isolated namespace
  };

  # Bind a service to the VPN namespace
  systemd.services.deluged = {
    after = [ "pia-vpn.service" ];
    bindsTo = [ "pia-vpn.service" ];
    serviceConfig.NetworkNamespacePath = "/var/run/netns/wg";
  };
```

**Test VPN IP from inside namespace:**
```bash
sudo ip netns exec wg curl -s https://ifconfig.me
```

**Backward compatibility:** Setting `namespace = null` (the default) uses the original behavior.
