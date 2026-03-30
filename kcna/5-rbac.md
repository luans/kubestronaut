# RBAC (Role-Based Access Control)

> **Ponto-chave:** Kubernetes **não gerencia** contas de usuário e grupo. Em vez disso, espera que certificados sejam criados e autorizados por uma Autoridade Certificadora (CA) externa. Usuários e grupos são representados por campos nos certificados, não como objetos no cluster.

---

## Identidades no Kubernetes

### Users (Usuários)
Indivíduos ou aplicações que interagem com o cluster — administradores, desenvolvedores ou sistemas automatizados. São **gerenciados externamente** ao Kubernetes; dentro do cluster, são representados apenas por uma string (ex: `alice` ou `alice@example.com`).

> Em certificados, o usuário é definido pelo campo **CN (Common Name)**.

### Groups (Grupos)
Também gerenciados fora do Kubernetes. Um grupo agrega múltiplos usuários e permite atribuir um conjunto de permissões de uma vez — todos os membros do grupo herdam essas permissões automaticamente.

> Em certificados, o grupo é definido pelo campo **O (Organisation)**.

### ServiceAccounts
Usados por **aplicações rodando dentro do próprio cluster** (não por humanos). Diferente de usuários e grupos, ServiceAccounts são **objetos Kubernetes** gerenciados pelo próprio cluster. Concedem as permissões necessárias para que um Pod interaja com a API do Kubernetes.

---

## Objetos RBAC

### Role e RoleBinding
Escopados a um **namespace** específico. Uma `Role` define o que pode ser feito; um `RoleBinding` associa essa Role a um usuário, grupo ou ServiceAccount dentro do namespace.

### ClusterRole
Recurso **não-namespaciado** — se aplica ao cluster inteiro e permite definir permissões sobre recursos em todos os namespaces. Também usado para recursos que não pertencem a nenhum namespace (ex: `nodes`, `persistentvolumes`).

### ClusterRoleBinding
Associa um `ClusterRole` a um usuário, grupo ou ServiceAccount com escopo **cluster-wide**.

---

## Resumo rápido

| Objeto               | Escopo      | Descrição                                              |
|----------------------|-------------|--------------------------------------------------------|
| Role                 | Namespace   | Define permissões dentro de um namespace               |
| RoleBinding          | Namespace   | Vincula uma Role a uma identidade no namespace         |
| ClusterRole          | Cluster     | Define permissões em todo o cluster                    |
| ClusterRoleBinding   | Cluster     | Vincula um ClusterRole a uma identidade cluster-wide   |

| Identidade      | Gerenciado por    | Campo no certificado | Caso de uso principal                            |
|-----------------|-------------------|----------------------|--------------------------------------------------|
| User            | Externo (certs)   | CN (Common Name)     | Humanos ou sistemas externos acessando o cluster |
| Group           | Externo (certs)   | O (Organisation)     | Conjunto de usuários com permissões em comum     |
| ServiceAccount  | Kubernetes        | —                    | Pods que precisam chamar a API do Kubernetes     |

---

## Comandos úteis

```bash
# Listar todos os ClusterRoleBindings com detalhes
kubectl get clusterrolebindings -o wide

# Criar um ClusterRole com todas as permissões
kubectl create clusterrole cluster-superhero --verb='*' --resource='*'

# Criar um ClusterRoleBinding associando o ClusterRole a um grupo
kubectl create clusterrolebinding cluster-superhero \
  --clusterrole=cluster-superhero \
  --group=cluster-superheroes
```

> **Dica de prova:** `--verb='*'` e `--resource='*'` concedem acesso total — equivalente a um superusuário. Em ambientes reais, sempre aplique o princípio do **menor privilégio**.
