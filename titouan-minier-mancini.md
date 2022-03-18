# Logbook - Titouan
___
## 09/02/2022 - Mercredi 9 Février

### Première tentative de création de composition pour K3S

Sur NixOS erreur avec NixOS-Compose au moment du build:
```bash
error getting attributes of path '/run/user/1000/nix-build-test-script.drv-0': Permission denied
```
Les permissions sur la machine sont celles de root, et l'option de virtualisation de Docker a été configurée dans le *configuration.nix* ainsi que la configuration des Cgroups comme indiqué dans la doc.
\
\
**Essais réalisés sur une machine Ubuntu avec Nix.**
\
\
Initialisation du projet nxc avec `nxc init`; l'option `-e --example` permet de lancer un exemple parmi ceux disponibles, est-ce qu'il existerait une option pour démarer avec un répertoire nommé ? Je ne la vois dans les options. A priori ça ne changerait que le nom de quelques variables *foo*, donc peu utile.
\
\
J'utilise l'option `-f --default-flavour` avec comme valeur `docker` en supposant que le build se fera par défault avec Docker.
\
\
Contenu du *nxc.json*: 
```json
{"composition": "composition.nix", "default_flavour": "nixos-test"}
```
Est-ce que l'option `-f` a bien marché ? Ne devrait-ce pas être Docker en *"default_flavour"* ?
\
\
Je ne comprends pas encore à quoi servent les fichiers dans le dossier *nix* dans *nxc*. Des fichiers internes au fonctionnement de nxc ? 
\
\
J'ai cloné le repo de nixpkgs (c'est lourd), je vais regarder le package de k3s pour m'inspirer (voire copier) et remplir le fichier *composition.nix* pour un premier essai.
\
\
Je cherche avec la commande `find -name k3s*`. Je vois que tous les fichiers de test qui nous intéressent sont dans *nixpkgs/nixos/tests*.
\
\
#### Je commence avec la composition d'une seule instance de K3S.

**Debuggage** - Erreurs lors du build:
```bash
error: The option 'virtualisation.diskSize' does not exist. Definition values: - In <unknown-file>: 4096
error: The option 'virtualisation.memorySize' does not exist. Definition values: - In <unknown-file>: 1536
```
J'ai commenté ces deux lignes pour le moment pour que le build fonctionne.
\
\
Quand je lance `nxc start -t` pour exécuter les test, ils échouent et la situation n'est pas très claire.
\
Avant que k3s ne démarre, ce retour de log s'affiche plusieurs fois:
```
store_manager_1 is up-to-date
starting docker-compose
(finished: starting docker-compose, in 0.01 seconds)
```
Cela perturbe beaucoup la lecture de la console.
\
\
L'erreur finale:
```
Error: command `${pauseImage} | k3s ctr image import -` failed (exit code 1), exception command `${pauseImage} | k3s ctr image import -` failed (exit code 1)
```
**Note**: la console reste ouverte (besoin de CTRL+C) après le `Removing network store_default` ce qui laisse penser que tous les processus n'ont pas été terminés
\
\
Je retire la partie incriminée du *test-script.py* (extrait de la *composition.nix* dans un fichier python) puis je rebuild et restart pour essayer d'obtenir une base qui fonctionne.
\
Ce n'est pas la seule commande qui échoue:
```
Error: command `k3s kubectl apply -f ${testPodYaml}` failed (exit code 1), exception command `k3s kubectl apply -f ${testPodYaml}` failed (exit code 1)
```
Je lance alors en mode intéractif pour essayer d'obtenir plus d'informations sur ces erreurs car je n'ai aucune idée de la raison pour laquelle la commande retourne une exception depuis le retour console.
\
\
Je lance:
```
nxc start
nxc connect
```
Je ne remarque rien de spécial mais je me rends tout de suite compte que l'erreur provient probablement des variables interpolées dans le script. L'ayant déplacé dans un autre fichier, les variables n'étaient évidemment plus accessibles (l'habitude que l'IDE me signale ce genre de problèmes...).
\
\
En relançant il n'y a plus d'erreur mais l'exécution semble bloquée.
\
Je retire des tests pour déterminer celui qui à l'air de bloquer.
\
\
**Note**: Le processus `nxc build -f docker` et `nxc start -t` à chaque changement est pénible. Un système de hot-reloading serait incroyable pour l'expérience développeur.
\
\
Le problème semble venir de l'importation de l'image Docker utilisée par le pod. 
\
Je vérifie qu'il n'y a pas de flux de données qui pourrait indiquer un long téléchargement, pas de données. 
\
Je cherche des informations sur la fonction `pkgs.dockerTools.streamLayeredImage` qui est utilisée pour construire l'image (depuis le store de Nix, je suis pas sûr de la forme qu'elle prend exactement). 
\
\
Le résultat qu'elle produit dans la commande où elle est utilisée est: `/nix/store/n61l0zwkjfxm58h382klh6z63hi4fyih-stream-pause` donc une dérivation il semblerait.
\
\
Je tente de la remplacer par une autre fonction: `pkgs.dockerTools.buildLayeredImage`.
\
La valeur devient: `/nix/store/qnw2bgs8hcvqwhij3p281xzx354ad0gn-pause.tar.gz`, je ne suis pas sûr de comprendre ce qu'il se passe dans le store et ce qu'est concrètement cette image Docker pour Nix.
\
La commande n'est pas valide avec cette valeur.
\
\
Je tente une autre fonction encore: `pkgs.dockerTools.buildImage`.
\
Résultat: `/nix/store/znx9k0x8mvhvi5vgjcl2iyl8d52f7iq7-docker-image-pause.tar.gz`, toujours pas une commande valide donc. 
\
\
Je pense que la fonction initiale est la bonne mais je ne m'explique pas le blocage. 
\
J'arrête les expériences pour aujourd'hui. Je demanderait demain lors de la réunion.
\
Ensuite j'essaierai de créer plusieurs instances.

___
## 10/02/2022 - Thursday 10th of February

### Follow-up on K3S composition

From the discussion with Jonathan, K3S might not work inside of Docker and thus it would be better to test it with nixos-test or Grid5000 flavours.
\
\
I first try to build the composition for nixos-test. 
\
There is an error from the pod, which cannot pull the image used in the test:
```log
"Failed to create sandbox for pod" err="rpc error: code = Unknown desc = failed to get sandbox image \"rancher/pause:3.1\": failed to pull image \"rancher/pause:3.1\": failed to pull and unpack image \"docker.io/rancher/pause:3.1\": failed to resolve reference \"docker.io/rancher/pause:3.1\": failed to do request: Head \"https://registry-1.docker.io/v2/rancher/pause/manifests/3.1\": dial tcp: lookup registry-1.docker.io: no such host" pod="kube-system/coredns-7448499f4d-pc4l4"
```
I then found out I actually needed to specify the right pause image for K3S, which by default is "docker.io/rancher/pause:3.1" (what we see in the error message). For some reason this endpoint is not accessible from inside the test nixos VM (or maybe the k3s server) and thus we import (load) the image with the command `k3s ctr image import <image location>`.
\
\
**Note on images**: Pause containers (using pause image) always exist inside a pod to hold the namespace for other containers inside. I wonder whether this is why we need to specify the location of this image or juste because we also run it anyway.
\
\
With this configuration all the tests seem to pass correctly.
\
\
Final composition file:
```nix
{ pkgs, ... }:
let
  imageEnv = pkgs.buildEnv {
    name = "k3s-pause-image-env";
    paths = with pkgs; [ tini (hiPrio coreutils) busybox ];
  };
  pauseImage = pkgs.dockerTools.streamLayeredImage {
    name = "test.local/pause";
    tag = "local";
    contents = imageEnv;
    config.Entrypoint = [ "/bin/tini" "--" "/bin/sleep" "inf" ];
  };
  testManifest = pkgs.writeText "test.yml" (builtins.readFile ./pod.yaml);
in
{
  nodes = {
    server = { pkgs, ... }: {
      environment.systemPackages = with pkgs; [ k3s gzip ];

      # k3s uses enough resources the default vm fails.
      virtualisation.memorySize = 1536;
      virtualisation.diskSize = 4096;

      services.k3s.enable = true;
      services.k3s.role = "server";
      services.k3s.package = pkgs.k3s;
      # Slightly reduce resource usage
      # services.k3s.extraFlags = "--no-deploy coredns,servicelb,traefik,local-storage,metrics-server --pause-image test.local/pause:local";
      services.k3s.extraFlags = "--pause-image test.local/pause:local";
    };
  };

  testScript = ''
    start_all()
    server.wait_for_unit("k3s")
    server.succeed("k3s kubectl cluster-info")
    server.succeed("${pauseImage} | k3s ctr image import -")
    server.succeed("k3s kubectl apply -f ${testManifest}")
    server.succeed("k3s kubectl wait --for 'condition=Ready' pod/test")
    server.succeed("k3s kubectl delete -f ${testManifest}")
    server.shutdown()
  '';
}

```

#### Updating the composition to add k3s agents to the cluster

The idea now is to spawn 3 agents (so on 3 nodes) appart from the original k3s server (equivalent to k8s control plane) (running on a third node). These 2 agents will run the same image as part of a deployment. To keep things simple we will pick a basic stateless application like a NodeJS server.
\
\
To spawn multiple similar nodes I took the function from Antoine's work on k3s.
\
\
First I would like to see if images are loaded automatically or if some further configuration is required (like the pause image).
\
\
There is an error when agents want to join the join by asking the server. I can't see the error in detail because the build is stuck looping on network/TLS handshake errors and I struggle finding the logs.
\
I tried this: `ls -ltr /nix/store | grep vm-test-run-unnamed.drv | head -1` to find the last derivation but all are dated of January 1970 and I don't know why.
\
\
I must find a way to debug this. The authentication token in K3S should have been correctly parametrized though.
\
\
After that I will have to investigate Docker flavour to check what was actually the problem. And then try to make this work on Grid5000.

___
## 11/02/2022 - Friday 11th of February

While debugging I realised that the pause image (the one used internally by K8S as I explained above) could not be found with the extra flag `--pause-image test.local/pause:local` until the image is imported with `${pauseImage} | k3s ctr image import -` which of course makes sense. However it means it should not have worked during the earlier phase. I am confused about the error I had with the pause image from Rancher registry which could not be retrieved from the network as now this seems to work just fine. Anyway, for now I will remove this flag as it makes sense to me.
\
\
**Note**: When using interactive mode and when I use `start_all()` the logging loops and makes the console ususable. I guess this is a K3S behavior.
```log
server # [  731.400222] k3s[636]: Trace[1285019316]: ---"Transaction committed" 45ms (16:50:00.576)
server # [  731.401222] k3s[636]: Trace[1285019316]: ---"Transaction prepared" 1ms (16:50:00.577)
server # [  731.403175] k3s[636]: Trace[1285019316]: ---"Transaction committed" 41ms (16:50:00.618)
server # [  731.407581] k3s[636]: Trace[1285019316]: ---"Transaction prepared" 5ms (16:50:00.624)
```
\
\
**Note**: The autocompletion inside the interactive shell is very useful and practicle. However is it visually overwrittend by the logging, making it unusable when there is a ton of things going on. 
\
\
**Note**: The layout of the interactive shell does not display all the content of a log line unless we input something (there is not line break to show the whole log).
\
\
**Note**: When entering a specific machine (ie when using `<machine>.shell_interact()`) there is still the logging from all the others machines and it's hard to figure we are indeed ssh-ed inside the target machine (not much feedback, and it's hidden by the massive amount of logs).
\
\
It's still hard to understand why agent nodes can't connect to the server:
```log
server # [   54.333449] refused connection: IN=eth1 OUT= MAC=52:54:00:12:01:04:52:54:00:12:01:01:08:00 SRC=192.168.1.1 DST=192.168.1.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=13535 DF PROTO=TCP SPT=40
server # [   54.383926] refused connection: IN=eth1 OUT= MAC=52:54:00:12:01:04:52:54:00:12:01:02:08:00 SRC=192.168.1.2 DST=192.168.1.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=37328 DF PROTO=TCP SPT=40
server # [   54.583736] refused connection: IN=eth1 OUT= MAC=52:54:00:12:01:04:52:54:00:12:01:03:08:00 SRC=192.168.1.3 DST=192.168.1.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=56937 DF PROTO=TCP SPT=50
agent1 # [   57.298687] k3s[634]: time="2022-02-11T16:35:10.242211136Z" level=error msg="failed to get CA certs: Get \"https://127.0.0.1:6444/cacerts\": context deadline exceeded (Client.Timeout "
agent2 # [   57.280328] k3s[634]: time="2022-02-11T16:35:10.180447284Z" level=error msg="failed to get CA certs: Get \"https://127.0.0.1:6444/cacerts\": context deadline exceeded (Client.Timeout "
agent3 # [   57.409267] k3s[633]: time="2022-02-11T16:35:10.312553482Z" level=error msg="failed to get CA certs: Get \"https://127.0.0.1:6444/cacerts\": context deadline exceeded (Client.Timeout "
```
**Question**: Is it possible to hide logs from a certain machine ? Something working like `server.hide_logs()`.

___
## 16/02/2022 - Wednesday 16th of February

I created a Git repo "projects" for Jonathan to see and check the K3S composition. 
\
When I try to build (`nxc build`) from the folder inside the Git repo there is this error: 
```log
Error: 'NoneType' object is not subscriptable, exception 'NoneType' object is not subscriptable
```
Is this linked to Git role in Flakes ?
\
\
Steps to reproduce:
```bash
# Clone the repo
ssh-agent bash -c 'ssh-add ~/.ssh/nixos/id_rsa; git clone git@gitlab.inria.fr:nixos-compose/projet-info5/projets.git'
# Move to the NixOS-Compose repo to enter the shell
cd nixos-compose
nix-shell
poetry shell
# Go back to the nxc project location
cd ../projets/k3s-multinode/
nxc build -f nixos-test-driver
```
**Output** (it's the same with `nxc build`):
```bash
(nixos-compose-jaQYbl6M-py3.9) cao@pilot:~/NixOS/projets/k3s-multinode$ nxc build -f nixos-test-driver

Error: 'NoneType' object is not subscriptable, exception 'NoneType' object is not subscriptable
```
\
**Note on nix-shell**: If I use `nix shell` instead of `nix-shell` it doesn't do anything:
```bash
☸  ✗  default 🔑 cao 💻 pilot 📂 /home/cao/NixOS/nixos-compose 📍 git: master ✗  
➜   nix shell 
warning: Git tree '/home/cao/NixOS/nixos-compose' is dirty
☸  ✗  default 🔑 cao 💻 pilot 📂 /home/cao/NixOS/nixos-compose 📍 git: master ✗  
``` 
And I can't get into poetry shell of course.
\
\
I keep going using another folder outside the Git repo (but with the same content).
\
\
It really seems like it is a problem of port or network low-level issue, where the agent machine is not authorized to talk to the server machine regardless of K3S.
\
When I try `curl -vk https://server:6443/cacerts` from the Agent1 I get a `refused connection` message on the server. 
\
\
I try to use `networking.firewall.allowedTCPPorts = [ 6443 ];` and now I get a `Failed to connect to proxy" error="dial tcp 10.0.2.15:6443: connect: connection refused`.
\
\
It is clearly better though, because now it's a K3S problem. I can fetch CA certificates from the server now.
\
Regardless of this error, K3S knows about all the nodes in the cluster: `k3s kubectl get node` shows all the agents, it seems to work.
\
\
I update my test to check that the cluster is indeed 4-node large with a one liner bash command:
```bash
[ $(k3s kubectl get node --no-headers=true | wc -l) -ne 4 ] && exit 1 || exit 0
```
**Note**: When building with nixos-test the build process blocks (`k3s.service: Failed with result 'exit-code'`) because it seems it starts th tests at the same time and the tests loop in the failed assertion. It works just fine with nixos-test-driver.
```bash
(nixos-compose-jaQYbl6M-py3.9) cao@pilot:~/NixOS/sandbox/k3s-multinode$ nxc build
Starting Build
[1/1/2 built, 0.0 MiB DL] building vm-test-run-unnamed: server # [   31.990977] systemd[1]: Failed to start k3s service.
```
The test on the number of nodes seems to need more than just `wait_for_unit("k3s")` to have the time to get to success status. However it is a bit hard to predict when it will be.

___
## 17/02/2022 - Thursday 17th of February

*We prepared a slideshow for the presentation tomorrow.*
\
\
I'm trying to sleep the test execution to see if that's really the problem. It seems like there is still a problem of connection between agents and the server though. Some logs seem to indicate the TLS handshake didn't succeed.
```log
agent2 # [   17.337377] k3s[634]: time="2022-02-17T17:07:26.165518243Z" level=warning msg="Cluster CA certificate is not trusted by the host CA bundle, but the token does not include a CA hash. Use the full token fro"
server # [   17.293843] k3s[633]: time="2022-02-17T17:07:26.185654238Z" level=info msg="Cluster-Http-Server 2022/02/17 17:07:26 http: TLS handshake error from 192.168.1.2:59166: remote error: tls: bad certificate"
server # [   17.416210] k3s[633]: time="2022-02-17T17:07:26.308015320Z" level=info msg="Cluster-Http-Server 2022/02/17 17:07:26 http: TLS handshake error from 192.168.1.3:42456: remote error: tls: bad certificate"
server # [   17.496241] k3s[633]: time="2022-02-17T17:07:26.387816244Z" level=info msg="Cluster-Http-Server 2022/02/17 17:07:26 http: TLS handshake error from 192.168.1.1:57958: remote error: tls: bad certificate"
```
I try to add openssl to the packages, a bit randomly trying to fix that.
\
I'm running out of space to build new tests. How can I free the space used by the builds until now ?
\
I tried this:
```bash
sudo rm -rf *nixos-test-driver-unnamed*
sudo rm -rf *vm-test-run-unnamed*
```
```bash
error: opening file '/nix/store/pbx5kbajxzp4hkhvsjj06rrg3k88amj8-nixos-test-driver-unnamed.drv': No such file or directory
```
And I still don't have space. According to my file explorer I still have empty space though.
\
I don't know how to free space nor rebuild the missing derivation.

___
## 25/02/2022 - Friday 25th of February

I am having a look at Kubernetes derivation, modules and tests in nixpkgs to start with.
\
Tests seem to require quite a lot of configuration (expectable from k8s) but are very well done and clearly written as examples in nixpkgs.
\
\
There are tests for some applications to run inside the cluster. As a sidenote it might be nice to run ELK as a monitoring solution for example inside Kubernetes at some point.
\
\
I keep the file *base.nix* that helps creating Kubernetes tests by handling all the basic configuration (control-plane, networking...).
\
All the *node* part is already done.
\
\
If I try to build as it is I get this error:
```bash
error: getting status of '/nix/store/pi7ac1kmas8zf8gkvm94ccngcp9ar75v-source/kubernetes': No such file or directory
```
Which seemingly means that somehow the Kubernetes derivation is not detected as something to install beforehand.
\
\
I try to update the functions parameters to fetch the rights pkgs as it is done with nxc.
\
\
Actually that was not a dependancy problem but something related to the git repository where the new folder was not added to staged files.
\
By changing the folder name the error changed to that new name.
```bash
error: getting status of '/nix/store/pi7ac1kmas8zf8gkvm94ccngcp9ar75v-source/hello': No such file or directory
```
I then did a `git add --all` to add the new folder to the git repo and the problem was gone.
\
\
Now I get other errors because I don't provide the right attributes in the composition (testScript...).
\
I update the `in` to directly provide those with `dns.singlenode.test` but I see I cannot use the attribute `test` because I removed the function `makeTest` in the *base.nix*.
\
I originally did so because I though it was what NixOS-Compose was doing, but now I to second-guess that decision.
\
I checked that function from the nixpkgs libraries and it mostly looks like it's what NixOS-Compose is doing, so I will just remove it and adapt the code to fit what I am used to do with NixOS-Compose. It might impact the ability to run mutlitple tests at once (because of the way it's coded in nixpkgs with the use of makeTest) but anyway, for now it's OK.
\
\
The build process starts but I get a new error:
```
error: builder for '/nix/store/yxk2ky1g6jrc4kmf6hv6jf354isfpqpy-vm-test-run-kubernetes-dns-singlenode.drv' failed with exit code 1;
       last 10 log lines:
       >   File "/nix/store/mjzqvkgh65jk6bmak4rn4hqlzi00wadp-nixos-test-driver/bin/.nixos-test-driver-wrapped", line 134, in retry
       >     if fn(False):
       >   File "/nix/store/mjzqvkgh65jk6bmak4rn4hqlzi00wadp-nixos-test-driver/bin/.nixos-test-driver-wrapped", line 509, in check_success
       >     status, output = self.execute(command)
       >   File "/nix/store/mjzqvkgh65jk6bmak4rn4hqlzi00wadp-nixos-test-driver/bin/.nixos-test-driver-wrapped", line 449, in execute
       >     self.shell.send(out_command.encode())
       > BrokenPipeError: [Errno 32] Broken pipe
       > cleaning up
       > killing machine1 (pid 6)
       > (0.00 seconds)
       For full logs, run 'nix log /nix/store/yxk2ky1g6jrc4kmf6hv6jf354isfpqpy-vm-test-run-kubernetes-dns-singlenode.drv'.
```
I check the logs.
\
The machine instantly dies.
\
The error seems to be:
```
machine1 # Could not access KVM kernel module: Permission denied
machine1 # qemu-system-x86_64: failed to initialize kvm: Permission denied
```

___
## 02/03/2022 - Wednesday 2nd of March

I keep trying to solve the permission issues with KVM by reading what Corentin Sueur wrote in his logbook becasue he had this problem too.
\
Proposed solutions do not seem to work as expected, I still have the problem.
\
\
I try to use the test driver and start the machine alone with `machine1.start()`.
\
It seems to work just fine.
\
\
I then start the test routine. Everything seem to work properly. All the tests pass.
\
I try again, still with the driver but without the interactive mode. I works fine too.
\
Yet, it won't work without the driver.
\
\
Considering it's a local problem, I'll move on using the driver for VM based experiments.

___
## 03/03/2022 - Thursday 3rd of March

I refactored the files in nixpkgs to fit better nixos-compose's structure.
\
This way I put all the general configuration in the composition.nix and moved everything relative to the experiment in itself (like the machines, tests and additional configuration required) in another file: experiment.nix. I though this way I could manage my experiment (K8S resources) and tinker the experiment-level configuration without bothering about the general setup of Kubernetes.
\
\
The multinode DNS test passes. I will now work on a first experiment on my own.
\
\
I update the tests to create deployments and services for 2 apps on a total of 4 machines.

___
## 07/03/2022 - Monday 7th of March

I updated the project repo to include 3 different projects on kubernetes:
- One simple, with only Kubernetes
- One richer, with Helm
- One for a full experiment with Istio
\
\
I started working on the richer one.
\
I try to deploy on Grid5000 to get better usability of the environment.
\
\
I update the composition not to need to use ip addresses.
\
When I try it on Docker I get this error:
```bash
warning: Git tree '/home/cao/NixOS/projets' is dirty
error: builder for '/nix/store/6d4crhkfc06xblpsahs457xb3pgxh641-gtk+-2.24.32.drv' failed with exit code 1;
       last 10 log lines:
       > configure: error: Package requirements (cairo-xlib >= 1.6) were not met:
       >
       > No package 'cairo-xlib' found
       >
       > Consider adjusting the PKG_CONFIG_PATH environment variable if you
       > installed software in a non-standard prefix.
       >
       > Alternatively, you may set the environment variables CAIRO_BACKEND_CFLAGS
       > and CAIRO_BACKEND_LIBS to avoid the need to call pkg-config.
       > See the pkg-config man page for more details.
       For full logs, run 'nix log /nix/store/6d4crhkfc06xblpsahs457xb3pgxh641-gtk+-2.24.32.drv'.
error: 1 dependencies of derivation '/nix/store/8k6p7cfnwwnz7l5rbvx1j2p0c118lzc2-lv2-1.18.2.drv' failed to build
error: 1 dependencies of derivation '/nix/store/xkp4nvs3203ab0f08fvsa5rsbkq8g06l-helm-0.9.0.drv' failed to build
error: 1 dependencies of derivation '/nix/store/1xsxv874xdvxqc8h7l2y6h76lmb5g22b-system-path.drv' failed to build
error: 1 dependencies of derivation '/nix/store/3b7m3hljiqjfrfzpbc3qx2cd50i4afjd-system-path.drv' failed to build
error: 1 dependencies of derivation '/nix/store/1gkix77vli4x5xw2x8mqyj19m83yfcpb-nixos-system-unnamed-21.05pre-git.drv' failed to build
error: 1 dependencies of derivation '/nix/store/8f82ffrw47xr2grwdsms6xchq7gha485-docker-compose.drv' failed to build
error: 1 dependencies of derivation '/nix/store/jwngb7r2m6xcnli317afyar80z2jvqb6-compose-info.json.drv' failed to build
Warning: Build return code: 1
```
It worked after adding `environment.noXlibs = false;` to the nodes.
\
\
It takes a lot of time to debug the composition because everytime I make a small change I have to restart the driver, setup the cluster and try the config.
\
\
There are actually quite a lot of issues when I deploy on Docker, showing that the Kubelet, etcd and other components are not working properly (things I could not see with the nixos-test-driver).

___
## 07/03/2022 - Monday 7th of March

Docker will not be an easy platform to deploy Kubernetes with NixOS. 
\
\
On Grid5000 it will require further configuration to setup a nfs for token exchange. 

___
## 08/03/2022 - Tuesday 8th of March

I have been working on making the simple Kubernetes environnement functional without specifying IP addresses.
___
## 09/03/2022 - Tuesday 9th of March

I managed apitoken exchange using the python environment in the testScript.
\
I try to deploy on g5K.
___
## 11/03/2022 - Friday 11th of March

I deployed on g5k. There is a problem with the controller-manager `HTTP probe failed with statuscode: 400`.
\
I must also create a script to encapsulate what I did previously in the testScript.