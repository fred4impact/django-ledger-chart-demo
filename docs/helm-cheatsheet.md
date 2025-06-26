# Helm Cheatsheet

A quick reference guide for common Helm commands and usage patterns.

---

## Helm Basics

### Add a Helm Repository
```
helm repo add <repo-name> <repo-url>
```

### Update Repositories
```
helm repo update
```

### Search for Charts
```
helm search hub <keyword>
helm search repo <keyword>
```

### Install a Chart
```
helm install <release-name> <chart> [flags]
```

### Upgrade a Release
```
helm upgrade <release-name> <chart> [flags]
```

### Uninstall (Delete) a Release
```
helm uninstall <release-name>
```

### List Releases
```
helm list [flags]
```

### Get Release Status
```
helm status <release-name>
```

### Get Release Values
```
helm get values <release-name> [flags]
```

### Rollback a Release
```
helm rollback <release-name> [revision]
```

---

## Chart Development

### Create a New Chart
```
helm create <chart-name>
```

### Lint a Chart
```
helm lint <chart-directory>
```

### Package a Chart
```
helm package <chart-directory>
```

### Template Rendering (Dry Run)
```
helm template <chart> [flags]
```

---

## Working with Values

### Override Values from File
```
helm install <release-name> <chart> -f <values.yaml>
```

### Override Values from CLI
```
helm install <release-name> <chart> --set key1=val1,key2=val2
```

---

## Debugging

### Debug Install/Upgrade
```
helm install <release-name> <chart> --debug --dry-run
```

### Show All Resources
```
helm get manifest <release-name>
```

---

## Repo Management

### List Repositories
```
helm repo list
```

### Remove a Repository
```
helm repo remove <repo-name>
```

---

## Useful Flags
- `--namespace <ns>`: Specify namespace
- `--set key=val`: Set values on the command line
- `-f <file.yaml>`: Specify values file
- `--dry-run`: Simulate an install/upgrade
- `--debug`: Show detailed output

---

## Resources
- [Helm Docs](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/) 