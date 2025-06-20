# unifi-network-application

![Version: 0.3.7](https://img.shields.io/badge/Version-0.3.7-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 9.2.87](https://img.shields.io/badge/AppVersion-9.2.87-informational?style=flat-square)

A Helm chart for Kubernetes

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Balazs Petrikovics | <bpetrikovics@gmail.com> |  |
| Gabor Pichner | <95gabor@gmail.com> |  |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| additionalVolumeMounts | list | `[]` |  |
| additionalVolumes | list | `[]` |  |
| affinity | object | `{}` |  |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.repository | string | `"linuxserver/unifi-network-application"` |  |
| image.tag | string | `""` |  |
| imagePullSecrets | list | `[]` |  |
| ingress.annotations | object | `{}` |  |
| ingress.hostname | string | `"unifi.local"` |  |
| ingress.redirect.annotations | object | `{}` |  |
| ingress.redirect.enabled | bool | `true` |  |
| mongodb.additionalVolumeMounts | list | `[]` |  |
| mongodb.additionalVolumes | list | `[]` |  |
| mongodb.adminpassword | string | `"replace-me"` |  |
| mongodb.adminuser | string | `"unifi-admin"` |  |
| mongodb.affinity | object | `{}` |  |
| mongodb.cacheSizeGB | string | `"0.25"` |  |
| mongodb.dbname | string | `"unifi"` |  |
| mongodb.dbname_stat | string | `"unifi_stat"` |  |
| mongodb.image | string | `"mongo"` |  |
| mongodb.imageTag | string | `"4.4"` |  |
| mongodb.nodeSelector | object | `{}` |  |
| mongodb.password | string | `"replace-me"` |  |
| mongodb.persistence.accessMode | string | `"ReadWriteOnce"` |  |
| mongodb.persistence.enabled | bool | `true` |  |
| mongodb.persistence.size | string | `"4Gi"` |  |
| mongodb.tolerations | list | `[]` |  |
| mongodb.username | string | `"unifi"` |  |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| persistence.accessMode | string | `"ReadWriteOnce"` |  |
| persistence.enabled | bool | `true` |  |
| persistence.size | string | `"4Gi"` |  |
| replicaCount | int | `1` |  |
| resources | object | `{}` |  |
| service.annotations | object | `{}` |  |
| service.loadBalancerIp | string | `nil` |  |
| service.type | string | `nil` |  |
| tolerations | list | `[]` |  |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)
