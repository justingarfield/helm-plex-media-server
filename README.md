# helm-plex-media-server

This repository contains the files I use to provision my Plex Media Server (PMS) container inside of Kubernetes using Helm.

## Assumptions

* You are running **Kubernetes 1.25** under **Microk8s w/ DNS addon**, on **Ubuntu Linux 22.04** [add link to Documentation](#)
* Your Media files are exposed via an **SMB Share**
* You don't care about using **Host Networking** _(will be swapping out for Bridged-mode eventually)_

## What's in the box?

| File | Description |
|-|-|
| `/csi-driver-smb/microk8s-values.yml` | Used when installing the _csi-driver-smb_ CSI to make it work with Microk8s. |
| `/persistent-volume-claims.yml`       | Collection of PersistentVolumeClaims used for PMS |
| `   - plex-data-pvc`                  | - Reserves its corresponding PersistentVolume (`plex-data-pv`) so that it will be pre-existing to pass into the K8s-at-home Helm Chart. |
| `   - media-pvc`                      | - Reserves its corresponding PersistentVolume (`media-pv`) so that it will be pre-existing to pass into the K8s-at-home Helm Chart. |
| `/persistent-volumes.yml`             | Collection of PersistentVolumes used for PMS |
| `   - plex-data-pv`                   | - Points to wherever you want to store your Plex Media Server data. |
| `   - media-pv`                       | - Points to wherever you host your Media files. |
| `/values.yml`                         | Overrides and preferences for the `k8s-at-home/plex` Helm chart. Do all your big configuration stuff here. |
| `   - TZ`                             | - Set the Pod/Container timezone |
| `   - env.PLEX_PREFERENCE_XX`         | - Override/Set a buttload of PMS settings up-front, so we can re-create / not have to go through Web UI |
| `   - hostNetwork`                    | - Set to true (for now, until Bridged Networking is solved) |
| `   - dnsPolicy`                      | - Set this to `ClusterFirstWithHostNet` to work with `hostNetwork` |
| `   - dnsConfig`                      | - Had to configure `ndots` to `1` in order for DNS lookups to function under Microk8s / CoreDNS / hostNetwork mode |
| `   - persistence.transcode`          | - Set `medium: Memory` to put `/transcode` mapped-volume into memory-backed tmpfs |

## Configuration / Usage

### Installing the SMB CSI

```shell
helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
helm install csi-driver-smb csi-driver-smb/csi-driver-smb --create-namespace --namespace csi-driver-smb --version v1.9.0 -f ./csi-driver-smb/microk8s-values.yml
```

### Create a Namespace for PMS

```shell
kubectl create namespace plex-media-server
```

### Configure SMB Credentials, PersistentVolumes, and PersistentVolumeClaims

```shell
kubectl -n plex-media-server create secret generic smbcreds --from-literal username=<SMB Share username> --from-literal password="<SMB Share password>"

kubectl create -f persistent-volumes.yml

kubectl -n plex-media-server create -f persistent-volume-claims.yml
```

### Provision PMS

Visit https://www.plex.tv/claim/, login, and get a Claims Token prepared.

```
helm install plex-media-server k8s-at-home/plex -n plex-media-server -f values.yml --set env.PLEX_CLAIM=<claims token here>
```

## Side Notes

* Quick Commands Reference
  * `helm delete plex-media-server -n plex-media-server`
  * `kubectl -n plex-media-server delete -f persistent-volume-claims.yml`
  * `kubectl delete -f persistent-volumes.yml`
  * `kubectl -n plex-media-server delete secret smbcreds`
  * `kubectl delete namespace plex-media-server`
  
* If you `sudo snap remove microk8s` to wipe the slate clean...
  * Be sure to `ip a` and check for calico/flannel related interfaces. Sometimes they don't all get cleaned up properly, and you need to `sudo ip link del <if name>` them
  * Be sure to cleanup any saved snapshots you no longer care about (`sudo snap saved` and `sudo snap forget`)
* Directories to be aware of
  * `plex-data-pv:Library/Application Support/Plex Media Server`
  * `plex-data-pv:Library/Application Support/Plex Media Server/Crash Reports`
  * `plex-data-pv:Library/Application Support/Plex Media Server/Logs`
  * `plex-data-pv:Library/Application Support/Plex Media Server/Preferences.xml`
* Found a buttload of PMS configuration/env var settings located at `http://127.0.0.1:32400/:/prefs` when the Web UI loads up. (Watch the Network / XHR tab for this)
* Found an interesting project to look into possibly: [kube-plex](https://github.com/munnerz/kube-plex)

## References

* [plex: Claim Code (Claims Token)](https://www.plex.tv/claim/)
* [DockerHub: Official PMS Docker Image](https://hub.docker.com/r/plexinc/pms-docker/)
* [GitHub: Official PMS Docker Image Repo](https://github.com/plexinc/pms-docker)
* [GitHub: k8s-at-home/charts repo](https://github.com/k8s-at-home/charts/blob/master/charts/stable/plex/values.yaml)
* [GitHub: k8s-at-home/library-charts](https://github.com/k8s-at-home/library-charts/blob/main/charts/stable/common/values.yaml)
* [csi-driver-smb](https://github.com/kubernetes-csi/csi-driver-smb)
* [CIFS Mount not performed despite logs indicating "mount succeeded"](https://github.com/kubernetes-csi/csi-driver-smb/issues/443)
* [How to reclaim your server after resetting your account password, if you can't access the server directly.](https://www.reddit.com/r/PleX/comments/wwfjqm/how_to_reclaim_your_server_after_resetting_your/)
* [Plexopedia: Server Capabilities](https://www.plexopedia.com/plex-media-server/api/server/capabilities/)
* [plex: Opening Plex Web App](https://support.plex.tv/articles/200288666-opening-plex-web-app/) _(mainly for the section **Opening the Plex Web App on a Device Other than the Server Itself**_)
* [plex: Advanced, Hidden Server Settings](https://support.plex.tv/articles/201105343-advanced-hidden-server-settings/)
* [plex: Troubleshooting Remote Access](https://support.plex.tv/articles/200931138-troubleshooting-remote-access/)
* [plex: Why am I locked out of Server Settings and how do I get in?](https://support.plex.tv/articles/204281528-why-am-i-locked-out-of-server-settings-and-how-do-i-get-in/)
* [plex: Cannot claim server](https://forums.plex.tv/t/cannot-claim-server/724877/38)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes)
* [Self-host your Media Center On Kubernetes with Plex, Sonarr, Radarr, Transmission and Jackett](https://kauri.io/#collections/Build%20your%20very%20own%20self-hosting%20platform%20with%20Raspberry%20Pi%20and%20Kubernetes/(58)-self-host-your-media-center-on-kubernetes-wi/)
* [PSA for Server Claiming: Check allowedNetworks](https://www.reddit.com/r/PleX/comments/wxa33r/psa_for_server_claiming_check_allowednetworks/)
* [Plex running in docker, can't find server](https://www.reddit.com/r/PleX/comments/ai9xyw/comment/eenf61e/)
* [plex-claim-server.sh](https://raw.githubusercontent.com/uglymagoo/plex-claim-server/master/plex-claim-server.sh)
* [claimpms.sh](https://github.com/ukdtom/ClaimIt/blob/master/linux/claimpms.sh)
