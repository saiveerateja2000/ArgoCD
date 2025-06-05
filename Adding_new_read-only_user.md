# Creating a New Read-Only User in Argo CD

This guide explains how to create a new user in Argo CD with limited (e.g., read-only) access using Argo CD's built-in RBAC (Role-Based Access Control).

---

## 📟 Prerequisites

* Argo CD is installed and accessible (UI or CLI).
* Access to the Argo CD namespace (typically `argocd`).
* `kubectl` access to the cluster.
* Basic knowledge of ConfigMaps in Kubernetes.

---

### 🛠️ Step 1: Enable Additional Accounts in `argocd-cm`

Edit the `argocd-cm` ConfigMap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Add your new user under the `accounts` section:

```yaml
data:
  accounts.devuser: |
    login
```

Save and exit. This enables the user to log in.

---

### ⚙️ Step 2: Define RBAC Permissions in `argocd-rbac-cm`

Edit or create the `argocd-rbac-cm` ConfigMap:

```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

Add the following content:

```yaml
data:
  policy.csv: |
    p, devuser, applications, get, *, allow
    p, devuser, clusters, get, *, allow
    p, devuser, projects, get, *, allow
  policy.default: role:readonly
  policy.matchMode: glob
  scopes: '[groups]'
```

This config gives `devuser` read-only access to applications, clusters, and projects.

> 📀 `policy.default: role:readonly` ensures all undefined users get read-only access. You can remove this line if you want to deny by default.

---

### 🔐 Step 3: Set a Password for the New User

Use a bcrypt-hashed password for the new user. An easy way to generate one is by using an online bcrypt generator:

🔗 Visit: [https://bcrypt-generator.com/](https://bcrypt-generator.com/)

1. Enter your desired password (e.g., `devpass123`).
2. Copy the generated bcrypt hash (should start with `$2a$`).

Then patch the ArgoCD secret:

```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "accounts.devuser.password": "<bcrypt-password-here>"
  }}'
```

> 📌 Ensure the password is a **bcrypt hash** (starts with `$2a$`). Replace `<bcrypt-password-here>` with the generated value.

---

### 🔄 Step 4: Restart ArgoCD Server

Apply the changes by restarting the ArgoCD server:

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

---

### 🧪 Step 5: Log In as the New User

Visit the Argo CD UI or use CLI:

```bash
argocd login <ARGOCD_SERVER> --username devuser --password <your-password>
```

You should now be able to log in and have **read-only access**.

---

### ✅ Example: Creating a Read-Only User `devuser`

Let’s say you want to create a user `devuser` who can only view apps, clusters, and projects.

1. Edit `argocd-cm` and add:

   ```yaml
   accounts.devuser: |
     login
   ```

2. Edit `argocd-rbac-cm` and add:

   ```yaml
   policy.csv: |
     p, devuser, applications, get, *, allow
     p, devuser, clusters, get, *, allow
     p, devuser, projects, get, *, allow
   policy.default: role:readonly
   policy.matchMode: glob
   scopes: '[groups]'
   ```

3. Generate a password hash:

   * Go to [https://bcrypt-generator.com/](https://bcrypt-generator.com/)
   * Enter `devpass123` and copy the hash.

4. Patch the secret:

   ```bash
   kubectl -n argocd patch secret argocd-secret \
     -p '{"stringData": {
       "accounts.devuser.password": "$2a$10$EXAMPLEHASH"
     }}'
   ```

5. Restart ArgoCD server:

   ```bash
   kubectl -n argocd rollout restart deployment argocd-server
   ```

6. Login using `devuser` and `devpass123` from the ArgoCD UI.

---

### 🔒 Optional: Granting More Fine-Grained Access

If you'd like `devuser` to sync applications or delete them:

```yaml
policy.csv: |
  p, devuser, applications, get, *, allow
  p, devuser, applications, sync, *, allow
  p, devuser, applications, delete, *, allow
```

To understand more verbs like `sync`, `delete`, `create`, refer to ArgoCD RBAC docs:
🔗 [https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

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
