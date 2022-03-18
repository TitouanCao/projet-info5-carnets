# Logbook - Corentin Humbert

## 09/02/2022 - Wednesday, 9 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills) (1-5):
  - First steps with Nix
  - Experimented with `nix repl`
- Started taking notes on Nix Pills on the following HackMD: https://hackmd.io/@Coko/HkJwMtRCK

---

## 10/02/2022 - Thursday, 10 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills) (6-8):
  - Wrote **derivations** using a bash builder (`builder.sh`) and `.nix` files
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)

---

## 11/02/2022 - Friday, 11 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills):
  - **Chapter 9:** Automatic Runtime Dependencies
  - **Chapter 10:** Developing with nix-shell
  - **Chapter 11:** Garbage Collector
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)

---

## 18/02/2022 - Friday, 18 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills):
  - **Chapter 12:** Automatic Runtime Dependencies
  - **Chapter 13:** Callpackage Design Pattern
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)
- Installed and tested `nixos-compose`

---

## 21/02/2022 - Monday, 21 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills):
  - **Chapter 14:** Override Design Pattern
  - **Chapter 15:** Nix Search Paths
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)
- Attempted to solve a build problem with a **k3s** composition on **Grid 5000** with Corentin Sueur

---

## 22/02/2022 - Tuesday, 22 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills):
  - **Chapter 16:** Nixpkgs Parameters
  - **Chapter 17:** Nixpkgs Overriding Packages
  - **Chapter 18:** Nix Store Paths
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)
- Met up with Corentin Sueur to look at yesterday's build problem. The problem was simply missing configurations in the **k3s** composition
- Downloaded [nixpkgs](https://github.com/NixOS/nixpkgs/archive/21.11.tar.gz) and started looking into **elk**
- Followed **nixos-compose** Grid5000 setup [guidelines](https://gitlab.inria.fr/nixos-compose/projet-info5/docs/-/blob/main/nix-g5k.md)

---

## 23/02/2022 - Wednesday, 23 February 2022

- Read and experimented with [Nix Pills](https://nixos.org/guides/nix-pills):
  - **Chapter 19:** Fundamentals of Stdenv
  - **Chapter 20:** Basic Dependencies and Hooks
- Took notes on [HackMD](https://hackmd.io/@Coko/HkJwMtRCK)
- Learned about [Nix Flakes](https://nixos.wiki/wiki/Flakes)

---

## 25/02/2022 - Friday, 25 February 2022

- Attempted to run the ELK test included in nixpkgs but my filesystem ran out of diskspace so I made some
- Encountered several problems with `nxc build` on **Grid 5000** (with the basic example):
  - I tried `nxc build` outside of the **nxc** directory which outputs the following error:
  ```bash
  Error: [Errno 2] No such file or directory: '/tmp/.flavours.json', exception [Errno 2] No such file or directory: '/tmp/.flavours.json'
  ```
  - I had forgotten to enable `flakes` in my **Grid 5000** nix config file:
  ```conf
  # /home/login/.config/nix/nix.conf
  experimental-features = nix-command flakes
  ```
- Updated flakes on local `nixos-compose` by running `nix flake update`

---

## 01/03/2022 - Tuesday, 1 March 2022

- While writing the **ELK** composition, I noticed that executing the `test-script` with the Docker flavour fails sometimes. The command seems to randomly succeed or fail with no apparent reason. The same problem appears on the `basic example` obtained with `nxc init`
- Steps to reproduce:
  - Build with the Docker flavour: `nxc build -f docker`
  - Run `nxc start -tf docker`
- In the end, I noticed three different scenarios (each scenario happens randomly when I try to start `test-script`):

  - **Scenario 1:** Creating `store_foo_1` fails:

  ```bash
  $ nxc start -tf docker
  Starting
  Use last build:
  /home/coko/Workspace/INFO5/s10/nixOS_project/nixos-compose/projets/elk/nxc/build/composition::docker
  run the test script
  additionally exposed symbols:
      foo,
      ,
      start_all, test_script, machines, driver, log, os, subtest, run_tests, join_all, retry, serial_stdout_off, serial_stdout_on, Machine
  foo: must succeed: true
  starting docker-compose
  (finished: starting docker-compose, in 0.06 seconds)
  Creating network "store_default" with the default driver
  Creating store_foo_1 ...
  foo: output:

  Error: command `true` failed (exit code 1), exception command `true` failed (exit code 1)
  Creating store_foo_1 ... error

  ERROR: for store_foo_1  Cannot start service foo: network store_default not found

  ERROR: for foo  Cannot start service foo: network store_default not found
  ERROR: Encountered errors while bringing up the project.
  ^C
  ```

  - **Scenario 2:** Creating does not fail but command `true` does not succeed:

  ```bash
  $ nxc start -tf docker
  ...
  (finished: starting docker-compose, in 0.01 seconds)
  Creating network "store_default" with the default driver
  Starting store_foo_1 ... done
  foo: output: OCI runtime exec failed: exec failed: container_linux.go:380: starting container process caused: exec: "bash": executable file not found in $PATH: unknown


  Error: command `true` failed (exit code 126), exception command `true` failed (exit code 126)
  Stopping store_foo_1 ... done
  Removing store_foo_1 ... done
  Removing network store_default
  ```

  - **Scenario 3:** Everything works:

  ```bash
  $ nxc start -tf docker
  ...
  (finished: starting docker-compose, in 0.01 seconds)
  Creating network "store_default" with the default driver
  Starting store_foo_1 ... done
  (finished: must succeed: true, in 6.30 seconds)
  (finished: run the test script, in 6.30 seconds)
  test script finished in 6.30s
  That's All Folk
  Stopping store_foo_1 ... done
  Removing store_foo_1 ... done
  Removing network store_default
  ```

- Encountered a new error stating container could not be found:

  ```bash
  ERROR: No such container: 669ec69cc81391185f249fcfe00bb661e94b8e2dff015a3eff992df81b0e0140

  Error: Command '['docker-compose', '-f', '/nix/store/7dmnidf2abbciqv73bv6w6vyv1l1sji5-docker-compose', 'ps', '--services', '--filter', 'status=running']' returned non-zero exit status 1., exception Command '['docker-compose', '-f', '/nix/store/7dmnidf2abbciqv73bv6w6vyv1l1sji5-docker-compose', 'ps', '--services', '--filter', 'status=running']' returned non-zero exit status 1.
  ```

  I tried running `docker-compose -f /nix/store/7dmnidf2abbciqv73bv6w6vyv1l1sji5-docker-compose ps` to see if any container was running. It shows that one container is up:

  ```bash
  $ docker-compose -f /nix/store/7dmnidf2abbciqv73bv6w6vyv1l1sji5-docker-compose ps
    Name                  Command               State   Ports
  ------------------------------------------------------------
  store_foo_1   /nix/store/5b5y5f6hn77pks4 ...   Up
  ```

---

## 02/03/2022 - Wednesday, 2 March 2022

- Yesterday's problem cause could not be found. The `nxc start -tf docker` sometimes fails unexcpectedly. When it fails, most of the time, the Docker container is left running and must be killed manually.
- Encountered build error with **ELK** composition because **elasticsearch** is not under a free license:

  ```bash
  error: Package ‘elasticsearch-6.8.3’ in /nix/store/2pmh2zsax3y73dbq7p1cvf5xn152zj8p-source/pkgs/servers/search/elasticsearch/6.x.nix:58 has an unfree license (‘elastic’), refusing to evaluate.

    a) To temporarily allow unfree packages, you can use an environment variable
      for a single invocation of the nix tools.

        $ export NIXPKGS_ALLOW_UNFREE=1

    b) For `nixos-rebuild` you can set
      { nixpkgs.config.allowUnfree = true; }
    in configuration.nix to override this.

    Alternatively you can configure a predicate to allow specific packages:
      { nixpkgs.config.allowUnfreePredicate = pkg: builtins.elem (lib.getName pkg) [
          "elasticsearch"
        ];
      }

    c) For `nix-env`, `nix-build`, `nix-shell` or any other Nix command you can add
      { allowUnfree = true; }
    to ~/.config/nixpkgs/config.nix.
  (use '--show-trace' to show detailed location information)
  ```

  None of the suggested options worked. I found other people online with the same issue:

  - https://discourse.nixos.org/t/flakes-with-unfree-licenses/9405/2

  - https://discourse.nixos.org/t/only-one-nixpkgs-in-a-flake-input-can-allow-unfree/8866

- Started working on the Project English poster with Titouan

---

## 04/03/2022 - Friday, 4 March 2022

- After trying multiple unsuccessful fixes with `NIXPKGS_ALLOW_UNFREE=1`, I found that there existed free variants for the whole stack:

  - elasticsearch6-oss for **Elasticsearch**
  - logstash6-oss for **Logstash**
  - kibana6-oss for **Kibana**

  There is no build problem when using these free packages. For now, I will give up on trying to build with `nxc` for unfree versions.

- Continued with the ELK composition:

  - Asked help to Corentin Sueur to solve a build error with the Docker flavour. The following line fixed the problem: `environment.noXlibs = false;`
  - I was unable to have the same `composition.nix` for `nxc build` and `nxc build -f docker`:

    - **Elasticsearch** requires at least 2060MB of memory to be able to run (this is because it runs on the JVM and the JVM needs enough memory to properly function)
    - Specifying `virtualisation.memorySize = 3000;` in the node configuration solves the problem for `nxc build`. However, this option is not recognized by Docker:

    ```bash
    $ nxc build -f docker
    Starting Build
    error: The option 'virtualisation.memorySize' does not exist. Definition values:
          - In '<unknown-file>': 3000
    (use '--show-trace' to show detailed location information)
    Warning: Build return code: 1
    ```

    So, in order to build with the Docker flavour, I need to omit this option and it builds successfully. But, that causes `nxc build` (without the Docker flavour) to fail because of the JVM problem mentioned earlier. In the end, this means that I need to have two different `composition.nix` files: one that works with the Docker flavour, and one that works with the default nixos flavour.

- I still have problems with `nxc start -tf docker` randomly failing, but I may have an idea of what is causing this:

  ```bash
  $ nxc start -tf docker
  ...
  starting docker-compose
  (finished: starting docker-compose, in 0.01 seconds)
  Creating network "store_default" with the default driver
  Creating store_one_1 ...
  ERROR: No such container: 3060e70fa188dc9791b216275565ff590946a5f5fb5aea49129184fe94036cca

  Error: Command '['docker-compose', '-f', '/nix/store/17qpz58ynzdzxxyjmiq5pr9c5r42p6k2-docker-compose', 'ps', '--services', '--filter', 'status=running']' returned non-zero exit status 1., exception Command '['docker-compose', '-f', '/nix/store/17qpz58ynzdzxxyjmiq5pr9c5r42p6k2-docker-compose', 'ps', '--services', '--filter', 'status=running']' returned non-zero exit status 1.
  ...
  $ docker ps
  CONTAINER ID   IMAGE                          COMMAND                  CREATED          STATUS         PORTS     NAMES
  ```

  When running `test-script`, it fails because it says it could not find container `3060e70...cca`. As soon as the script finished with **exit status 1**, I ran `docker ps` and unsurprisingly, there was no container running. However, after waiting a few more seconds and using `docker ps`, I got the following:

  ```bash
  $ docker ps
  CONTAINER ID   IMAGE                          COMMAND                  CREATED          STATUS         PORTS     NAMES
  3060e70fa188   nxc-docker-base-image:latest   "/nix/store/h325pf1l…"   17 seconds ago   Up 2 seconds             store_one_1
  ```

  There is one container running and its truncated id matches the container created by `nxc start -tf docker`. If we look at the **CREATED** and **STATUS** times obtained with `docker ps`, we can see that the container was created 17 seconds ago (most likely when command `nxc start -tf docker` was ran) but that it has only been running for the past 2 seconds. This led me to believe that the problem was that the `nxc start -tf docker` command did not wait long enough for the container to start and thus failed because it could not find it.

  Since my host OS is installed on a HDD, I am often met with very long boot times (5-10 minutes before system is fully responsive) and applications do also take some time to launch. My guess is that sometimes the Docker container can start in time and the script can execute successfully. While, on some other times, the Docker container takes too long to start, causing the script to fail.
