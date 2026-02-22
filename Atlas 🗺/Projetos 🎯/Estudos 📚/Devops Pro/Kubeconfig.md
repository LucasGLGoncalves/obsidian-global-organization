Um "cheatsheet" bem direto de **kubeconfig**: ver, trocar contexto, trocar arquivo via **KUBECONFIG**, e **merge/flatten** de 2 configs em um único.
## Local padrão do kubeconfig

```bash
# Linux/macOS
~/.kube/config

# Windows (PowerShell)
$env:USERPROFILE\.kube\config
```

---

## Ver o que existe no kubeconfig

```bash
# Mostrar qual contexto está em uso agora
kubectl config current-context

# Listar contextos
kubectl config get-contexts

# Listar clusters e users cadastrados
kubectl config get-clusters
kubectl config get-users

# Ver o arquivo inteiro "bonitinho"
kubectl config view

# Ver em YAML bruto, incluindo dados embutidos (certs, etc.)
kubectl config view --raw
```

Dica rápida (filtrar só nomes):

```bash
kubectl config get-contexts -o name
```

---

## Trocar de contexto (context switch)

```bash
# Trocar para um contexto existente
kubectl config use-context NOME_DO_CONTEXTO
```

Criar/editar contexto (útil quando você quer montar manualmente):

```bash
kubectl config set-context MEUCTX --cluster=MEUCLUSTER --user=MEUUSER --namespace=MEUNS

# Definir namespace padrão do contexto atual
kubectl config set-context --current --namespace=meu-namespace
```

Renomear ou apagar contexto:

```bash
kubectl config rename-context CONTEXTO_ANTIGO CONTEXTO_NOVO
kubectl config delete-context CONTEXTO_QUE_VAI_SAIR
```

---

## Trocar o arquivo de config (KUBECONFIG e --kubeconfig)

### Opção 1: por comando (não muda seu terminal)

```bash
kubectl --kubeconfig /caminho/clusterA.yaml get ns
```

### Opção 2: por sessão (muda o terminal atual)

Linux/macOS:

```bash
export KUBECONFIG=/caminho/clusterA.yaml
kubectl config current-context
```

PowerShell:

```powershell
$env:KUBECONFIG="C:\caminho\clusterA.yaml"
kubectl config current-context
```

Voltar para o padrão:  
Linux/macOS:

```bash
unset KUBECONFIG
```

PowerShell:

```powershell
Remove-Item Env:KUBECONFIG
```

Ver qual kubeconfig está em uso:

```bash
echo $KUBECONFIG              # Linux/macOS
echo $env:KUBECONFIG          # PowerShell
```

---

## Merge de 2 kubeconfigs e gerar um unificado (flatten)

### Jeito recomendado: usando KUBECONFIG com lista de arquivos

Linux/macOS:

```bash
export KUBECONFIG=~/.kube/config:/caminho/clusterB.yaml

# Conferir o resultado do merge
kubectl config get-contexts

# Gerar um arquivo unificado com tudo "flatten"
kubectl config view --merge --flatten --raw > ~/.kube/config-unificado.yaml
```

PowerShell (separador é `;`):

```powershell
$env:KUBECONFIG="$HOME\.kube\config;C:\caminho\clusterB.yaml"

kubectl config get-contexts
kubectl config view --merge --flatten --raw | Out-File -Encoding utf8 "$HOME\.kube\config-unificado.yaml"
```

Depois, para usar o unificado:

```bash
export KUBECONFIG=~/.kube/config-unificado.yaml     # Linux/macOS
```

```powershell
$env:KUBECONFIG="$HOME\.kube\config-unificado.yaml" # PowerShell
```

### Forma ensinada no DEVOPS PRO para unir dois config.yaml em um único

```Shell
KUBECONFIG=~/.kube/configA.yaml:~/.kube/configB.yaml kubectl config view --flatten > merge-config.yaml
```

Desta forma você aplica como variável de ambiente "KUBECONFIG=" o resultado do "config view --flatten" que foi salvo dentro de um novo yaml, tudo em um único comando.

Podendo sobrescrever o config padrão seguindo com o comando

```shell
cp ~/.kube/merge-config.yaml ~/.kube/config
```


---

## Minify vs Flatten (diferença rápida)

```bash
# minify: mostra só o que o contexto atual precisa (recorta o resto)
kubectl config view --minify

# flatten: embute referências externas e deixa tudo autocontido (bom para "portar" config)
kubectl config view --flatten --raw
```

Se você quer um arquivo “só do contexto atual” e portátil:

```bash
kubectl config view --minify --flatten --raw > kubeconfig-so-este-contexto.yaml
```

---

## Diagnóstico rápido de “estou no cluster certo?”

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

---

## (Opcional) Atalho de qualidade de vida: kubectx/kubens

Se você usa bastante troca de contexto/namespace, vale instalar `kubectx` e `kubens`:

```bash
kubectx                 # lista contextos
kubectx NOME            # troca contexto
kubens                  # lista namespaces
kubens NAMESPACE        # troca namespace
```



#DevOps #Estudos #Tecnologia #Recorrente #Dicas #Comandos #Linux