# Avancée et coordination d'équipe

## Mardi 8 février 2022

- Terminé les NixPills, connaissance du langage plus approfondie mais toujours une compréhension du fonctionnement assez basique
- Test sur l'exemple du Readme du depot `nixos-compose`. Des gros soucis lors du build (`nxc build`):

    ```s
    foo: must succeed: true
    foo: waiting for the VM to finish booting
    foo: starting vm
    foo # Formatting '/build/vm-state-foo/foo.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=536870912 lazy_refcounts=off refcount_bits=16
    foo # Could not access KVM kernel module: Permission denied
    foo # qemu-system-x86_64: failed to initialize kvm: Permission denied
    foo: QEMU running (pid 6)
    foo: connected to guest root shell
    ```

    Il semblerait que ce soit un problème de permissions, pourtant je suis bien dans le groupe kvm.

- Le build fonctionne avec `-f docker` cependant il ne peux pas start, encore pour une raison de droit (lors d'une création de dossier il indique que le lecteur est en lecture seule).
- A la recherche de solutions à ce souci

## Mercredi 9 février 2022

- Résolution du problème de build; les sites suivants peuvent aider en cas de soucis:
  - En cas de problèmes de droits: 
    : <https://www.dedoimedo.com/computers/kvm-permission-denied.html>
  - pour l'installation de KVM:
    : <https://help.ubuntu.com/community/KVM/Installation>
  - En cas de soucis lors du lancement pour lancer le build créé avec docker (`nxc build -f docker`)
    : Si docker est installé avec le gestionnaire de paquet snap, cela peut poser soucis. L'installer selon depuis dépot plutot <https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository>

## Jeudi 10 février 2022

- Switching the daily report to english as requested by the teachers
- Tried to buil my first derivations, using what Antoine used during his previous internship
- Meeting with the team and teachers the morning
- https://github.com/justinwoo/nix-shorts/tree/master/posts

## Vendredi 11 février 2022

- Tried to use and undersand k3s
- Titouan helped me since I didn't fully understood
- After this help I now undersand better the packages system

## Monday February 14th 2022

- working as suggested by Titouan: adding nodes (server+agents) on a nig configuration.
- Usage of the flavor `nxc start -I -f nixos-test-driver` to connect to the running build
- Tring to run the build lead me to a better understanding of the global functionning of nixos.
- Nothing more to add, things are still shady in my head by from day to day it gets better. Frequent meeting are welcome since it allow me to clarify some aspects. The one from this morning was useful but I should have prepared more questions. I think I understood where to find the documentation but still, I'm not an expert on k3s nor kubernetes, so I'm still working on it.
- Also I tried to practice a bit on g5k, my account was retired, again.

## Wednesday February 16th 2022

- continuing the work of the previous day. With the following configuration (really close to hat Titouan did) I got the error below.

```nix
{ pkgs, ... }:
let
  testManifest = pkgs.writeText "test.yml" (builtins.readFile ./pod.yaml);
  k3sToken = "df54383b5659b9280aa1e73e60ef78fc";
in
{
  nodes = {
    server = { pkgs, ... }: {
      environment.systemPackages = with pkgs; [ k3s gzip ];

      networking.firewall.allowedTCPPorts = [ 6443 ];
      # k3s uses enough resources the default vm fails.
      virtualisation.memorySize = 1536;
      virtualisation.diskSize = 4096;

      services.k3s.enable = true;
      services.k3s.role = "server";
      services.k3s.package = pkgs.k3s;
      # Slightly reduce resource usage
      # services.k3s.extraFlags = "--no-deploy coredns,servicelb,traefik,local-storage,metrics-server --pause-image test.local/pause:local";
      services.k3s.extraFlags = "--agent-token ${k3sToken}  --disable-network-policy --cluster-cidr 10.24.0.0/16 --flannel-backend=host-gw";
    };
    # --advertise-address xxx --flannel-backend=none --disable traefik

    agent = { pkgs, ...}:{
      environment.systemPackages = with pkgs; [ k3s gzip ];

      # k3s uses enough resources the default vm fails.
      virtualisation.memorySize = 1536;
      virtualisation.diskSize = 4096;

      services.k3s.enable = true;
      services.k3s.package = pkgs.k3s;

      services.k3s.role = "agent";
      services.k3s.serverAddr = "https://10.0.0.10:6443";
      services.k3s.tokenFile = "/var/lib/rancher/k3s/server/node-token";
      services.k3s.token = k3sToken;
      # services.k3s.docker = True;
      # services.k3s.configPath = types.path;
      # Slightly reduce resource usage
      # services.k3s.extraFlags = "--no-deploy coredns,servicelb,traefik,local-storage,metrics-server --pause-image test.local/pause:local";
      services.k3s.extraFlags = "--node-label --node-name test-node";
    };
  };

  testScript = ''
    start_all()
    server.wait_for_unit("k3s")
    agent.wait_for_unit("k3s")
    agent.succeed("kubectl cluster-info dump")
    server.succeed("k3s kubectl cluster-info")
    agent.succeed("k3s kubectl cluster-info")
    server.fail("sudo -u noprivs k3s kubectl cluster-info")
    server.succeed("[ $(k3s kubectl get node --no-headers=true | wc -l) -ne 4 ] && exit 1 || exit 0")

    server.shutdown()
  '';
}
```

The error: 

```s
server # [   58.530392] k3s[653]: E0216 16:11:35.375152     653 kubelet.go:2211] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
agent # [   59.382338] k3s[632]: time="2022-02-16T16:11:36.250115893Z" level=error msg="failed to get CA certs: Get \"https://127.0.0.1:6444/cacerts\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"
```

after using the flags for the server `--flannel-backend=host-gw` we now get the error in loop (like hundred of times per sec): 

```s
server # [  179.170448] k3s[659]: Trace[566009550]: ---"About to apply patch" 36ms (16:59:00.814)
```

## Thursday February 17th 2022

- invastigating on the previous error

for the server
`failed to get sandbox image \"rancher/pause:3.1\": failed to pull image \"rancher/pause:3.1\"`

`agent # [   44.575437] k3s[683]: time="2022-02-17T17:23:07.430981243Z" level=error msg="failed to get CA certs: Get \"https://127.0.0.1:6444/cacerts\": read tcp 127.0.0.1:45410->127.0.0.1:6444: read`

## Monday February 21th 2022

- deploying k3d on nix on g5k to check if it works. It doesn't.
- Meeting with Corentin to disscuss about possible issues, misconfiguration or solutions without any solution.
- Still the same problem of certificates and ip adresses
- The library closed today at 5pm

## Tuesday February 22th

- by looking at Titouan's work, I havent alowed the port `6443` on the client and the server adress is hard coded on the client side, but `"https://server:6443"` is in fact also accepted. This solved the problem of certificates and most connexions problems in the cluster. Jonathan said Titouan's version is working on g5k. That's why I took a look at it, his version is almost the same as mine
- Jonathan tip: use `networking.firewall.enable =false`
- K3s is now functionning as expected
