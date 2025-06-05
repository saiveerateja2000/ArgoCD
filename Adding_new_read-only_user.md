# Creating a New Read-Only User in Argo CD

This guide explains how to create a new user in Argo CD with limited (e.g., read-only) access using Argo CD's built-in RBAC (Role-Based Access Control).

---

## 📟 Prerequisites

* Argo CD is installed and accessible (UI or CLI).
* Access to the Argo CD namespace (typically `argocd`).
* `kubectl` access to the cluster.
* Basic knowledge of ConfigMaps in Kubernetes.

---

## 👤 Step 1: Enable Additional User Authentication

Edit the `argocd-cm` ConfigMap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Under `data`, add or update the `accounts.devuser` key:

```yaml
data:
  accounts.devuser: login
```

> ✅ This creates a user named `devuser` who is allowed to log in.

Save and exit.

---

## 🔐 Step 2: Set a Password for the New User

Use the `argocd` CLI to set the password:

```bash
argocd account update-password --account devuser
```

You will be prompted for the new password.

---

## 📜 Step 3: Configure RBAC for the New User

Edit the `argocd-rbac-cm` ConfigMap:

```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

Add the following to define read-only access for `devuser`:

```yaml
data:
  policy.csv: |
    p, devuser, applications, get, *, allow
    p, devuser, clusters, get, *, allow
    p, devuser, projects, get, *, allow
  policy.default: role:readonly
```

Explanation of permissions:

| Action       | Meaning                   |
| ------------ | ------------------------- |
| applications | Read Argo CD applications |
| clusters     | Read cluster info         |
| projects     | Read Argo CD project info |

---

## 🔄 Step 4: Restart Argo CD Server (Optional)

In some versions, you may need to restart Argo CD server for changes to apply:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

---

## 🧪 Step 5: Log In as the New User

Visit the Argo CD UI or use CLI:

```bash
argocd login <ARGOCD_SERVER> --username devuser --password <your-password>
```

You should now be able to log in and have **read-only access**.

---

## 📛 Notes

* This **does not** affect the default `admin` user.
* You can define multiple users by repeating the process with different names.
* For advanced setups, you can also integrate with SSO (OIDC, LDAP, etc.).

---

## ✅ Summary

| Component        | What You Did                         |
| ---------------- | ------------------------------------ |
| `argocd-cm`      | Declared a new user                  |
| `argocd-rbac-cm` | Assigned RBAC policy                 |
| CLI or UI        | Set the user password & tested login |

Now you've successfully created a new Argo CD user with read-only permissions! 🎉
