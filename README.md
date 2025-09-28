1. Install docker, kubernetes, kind i helm according to official guidelines
https://docs.docker.com/engine/install/ubuntu/
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
https://kind.sigs.k8s.io/docs/user/quick-start/
https://helm.sh/docs/intro/install/

2. Create a kind cluster by "kind create cluster --config kind-config.yaml", but before create kind-config.yaml containing:
"
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.21.1
"

3. Remember about chartmuseum, run "./install_common_templates_to_helm.sh", and create RIC like in the vimeo video (version G!) (https://vimeo.com/867704832) by running "./install -f ../RECIPE_EXAMPLE/example_recipe_oran_g_release.yaml". Ensure that each pod is "Running", by command "kubectl get pods -n ricplt". If pods are stuck in "PodInitialiazing" or "ContainerCreating", run "./uninstall" and again "./install -f ../RECIPE_EXAMPLE/example_recipe_oran_g_release.yaml", a after a few such attempts each pod should be "Running".

4. Do steps from section "Setup xapp installation requirements" from https://github.com/JaykobJ/oran-xapp-dev-platform/blob/master/README.md

5. Do steps from section "Install ns-O-RAN" from https://github.com/JaykobJ/oran-xapp-dev-platform/blob/master/README.md 
BUT
Change Dockerfile into:

"
#==================================================================================
#	Copyright (c) 2022 Northeastern University
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#	   http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
#==================================================================================

FROM wineslab/o-ran-sc-bldr-ubuntu18-c-go:9-u18.04 as buildenv

ARG log_level_e2sim=3
# log_level_e2sim = 0 ->  LOG_LEVEL_UNCOND   0
# log_level_e2sim = 1 -> LOG_LEVEL_ERROR     1
# log_level_e2sim = 2 -> LOG_LEVEL_INFO      2
# log_level_e2sim = 3 -> LOG_LEVEL_DEBUG     3

# Install E2sim
RUN mkdir -p /workspace
RUN apt-get update && apt-get install -y build-essential git cmake libsctp-dev autoconf automake libtool bison flex libboost-all-dev

WORKDIR /workspace

RUN git clone -b develop https://github.com/wineslab/ns-o-ran-e2-sim /workspace/e2sim

RUN mkdir /workspace/e2sim/e2sim/build
WORKDIR /workspace/e2sim/e2sim/build
RUN cmake .. -DDEV_PKG=1 -DLOG_LEVEL=${log_level_e2sim}

RUN make package
RUN echo "Going to install e2sim-dev"
RUN dpkg --install ./e2sim-dev_1.0.0_amd64.deb
RUN ldconfig

WORKDIR /workspace

# Install ns-3
RUN apt-get install -y g++ python3 qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools

# Add this after apt-get update
RUN apt-get install -y gcc-8 g++-8

# Update symlinks to use gcc-8/g++-8 by default
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 80 && \
    update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-8 80

#RUN git clone -b release https://github.com/wineslab/ns-o-ran-ns3-mmwave /workspace/ns3-mmwave-oran
RUN git clone -b master https://github.com/wineslab/ns-o-ran-ns3-mmwave /workspace/ns3-mmwave-oran
RUN git clone -b master https://github.com/o-ran-sc/sim-ns3-o-ran-e2 /workspace/ns3-mmwave-oran/contrib/oran-interface

WORKDIR /workspace/ns3-mmwave-oran
#RUN ./waf configure --enable-tests --enable-examples
RUN ./ns3 configure --enable-tests --enable-examples
#RUN ./waf build
RUN ./ns3 build


WORKDIR /workspace

CMD [ "/bin/sh" ]
" (changes are from "# Add this after apt-get update" to "RUN ./ns3 build" and in "ARG log_level_e2sim=3" instead of 3 there was 2 - inspired by a tutorial on "openran gym": https://openrangym.com/tutorials/ns-o-ran)

And don't do:

"
Export image as .tar

docker save -o ~/ml_ric/nsoran/ns3.tar ns3

Import image into containerd in kubernets namespace

ctr -n=k8s.io images import ns3.tar
"

but instead of that use: "kind load docker-image ns3:latest", then as in that github readme create and apply a yaml file.

6. Do steps from section "Install KPIMON xapp" from https://github.com/JaykobJ/oran-xapp-dev-platform/blob/master/README.md
BUT
before launching launch_app.sh, do (under ################: change 172.18.0.3 to your IP address taken from "docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' registry"):

# show networks (to spot the kind network)
docker network ls #dodatek

# show which networks the control-plane container is on
docker inspect -f '{{json .NetworkSettings.Networks}}' kind-control-plane | jq . #dodatek

# example if network name is "kind"
docker network connect kind registry

#3) Update xapp-descriptor/config.json to use as "registry" IP the numeric IP from this command:
docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' registry

################

# 1) create a small containerd patch file on host
cat > /tmp/containerd-registry-patch.toml <<'EOF'
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."172.18.0.3:5000"]
  endpoint = ["http://172.18.0.3:5000"]
EOF

# 2) copy it into the kind control-plane container
docker cp /tmp/containerd-registry-patch.toml kind-control-plane:/tmp/patch.toml

# 3) backup the current config and append the patch
docker exec -it kind-control-plane bash -lc "cp /etc/containerd/config.toml /etc/containerd/config.toml.bak && cat /tmp/patch.toml >> /etc/containerd/config.toml"

#If 3) returns an error, do 3.1), 3.2), 3.3).

# 3.1) Show the host file, to be sure
cat /tmp/containerd-registry-patch.toml
# 3.2) Append the host file directly into the container's config
docker exec -i kind-control-plane bash -lc "cp /etc/containerd/config.toml /etc/containerd/config.toml.bak && cat >> /etc/containerd/config.toml" < /tmp/containerd-registry-patch.toml

# 3.3) Verify the appended part is present
docker exec -it kind-control-plane bash -lc "grep -A3 '172.18.0.3:5000' /etc/containerd/config.toml || echo 'mirror entry not found'"

# 4) restart the kind control-plane container so containerd reloads config
docker restart kind-control-plane

# 5) wait for node Ready (takes a few seconds)
kubectl get nodes -o wide

# 6) test from the kind network that the registry is reachable via HTTP
docker run --rm --network kind curlimages/curl -fsS -o /dev/null -w "HTTP %{http_code}\n" http://172.18.0.3:5000/v2/ || docker run --rm --network kind curlimages/curl -v http://172.18.0.3:5000/v2/

7. Enter into ns-3-pod by "kubectl exec -it -n ricplt ns-3-pod -- /bin/bash" (or from the same terminal: install "apt update && apt install -y dbus-x11" and run "gnome-terminal -- bash -c "kubectl exec -it -n ricplt ns-3-pod -- /bin/bash"") and run a simulation with your e2TermIp (to learn e2TerpIp, run "kubectl get pods -n ricplt -o wide"): run "cd ns3-mmwave-oran/" and then run "NS_LOG="RicControlMessage" ./ns3 run "scratch/scenario-zero.cc --simTime=15 --e2TermIp={Put here your e2TermIp}""

8. Enter into kpimon (xApp) (learn an exact pod name from "kubectl get pods -n ricxapp") "kubectl exec -it -n ricxapp {an exact pod name} -- /bin/bash" (or from the same terminal: install "apt update && apt install -y dbus-x11" and run "gnome-terminal -- bash -c "kubectl exec -it -n ricxapp {an exact pod name} -- /bin/bash"") and run: ./kpimon -f /opt/ric/config/config-file.json

After the above steps, I still get this error:

root@mati-laptop:/home/mati/oran1/kpimon# kubectl logs -n ricplt deployment-ricplt-e2term-alpha-659f47f757-dvbfh | tail -n 15
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43726 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43086 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43729 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43660 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43088 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43994 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43884 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43886 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43221 open=0 succ=0 fail=0 (hard=0 soft=0)
1759090578880 23/RMR [INFO] sends: ts=1759090578 src=service-ricplt-e2term-rmr-alpha.ricplt:38000 target=10.244.0.9:43999 open=0 succ=0 fail=0 (hard=0 soft=0)
{"ts":1759090651172,"crit":"ERROR","id":"E2Terminator","mdc":{"PID":"123167892817664","POD_NAME":"deployment-ricplt-e2term-alpha-659f47f757-dvbfh","CONTAINER_NAME":"container-ricplt-e2term","SERVICE_NAME":"RIC_E2_TERM","HOST_NAME":"kind-control-plane","SYSTEM_NAME":"SEP"},"msg":"Error 2 Decoding (unpack) E2AP PDU from RAN : "}
{"ts":1759090651172,"crit":"ERROR","id":"E2Terminator","mdc":{"PID":"123167892817664","POD_NAME":"deployment-ricplt-e2term-alpha-659f47f757-dvbfh","CONTAINER_NAME":"container-ricplt-e2term","SERVICE_NAME":"RIC_E2_TERM","HOST_NAME":"kind-control-plane","SYSTEM_NAME":"SEP"},"msg":"Error 2 Decoding (unpack) E2AP PDU from RAN : "}
{"ts":1759090651174,"crit":"ERROR","id":"E2Terminator","mdc":{"PID":"123167892817664","POD_NAME":"deployment-ricplt-e2term-alpha-659f47f757-dvbfh","CONTAINER_NAME":"container-ricplt-e2term","SERVICE_NAME":"RIC_E2_TERM","HOST_NAME":"kind-control-plane","SYSTEM_NAME":"SEP"},"msg":"Error 2 Decoding (unpack) E2AP PDU from RAN : "}
{"ts":1759090651176,"crit":"ERROR","id":"E2Terminator","mdc":{"PID":"123167892817664","POD_NAME":"deployment-ricplt-e2term-alpha-659f47f757-dvbfh","CONTAINER_NAME":"container-ricplt-e2term","SERVICE_NAME":"RIC_E2_TERM","HOST_NAME":"kind-control-plane","SYSTEM_NAME":"SEP"},"msg":"Error 2 Decoding (unpack) E2AP PDU from RAN : "}
{"ts":1759090651176,"crit":"ERROR","id":"E2Terminator","mdc":{"PID":"123167892817664","POD_NAME":"deployment-ricplt-e2term-alpha-659f47f757-dvbfh","CONTAINER_NAME":"container-ricplt-e2term","SERVICE_NAME":"RIC_E2_TERM","HOST_NAME":"kind-control-plane","SYSTEM_NAME":"SEP"},"msg":"Error 2 Decoding (unpack) E2AP PDU from RAN : "}



