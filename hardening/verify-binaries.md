# Verificar Binários da Plataforma

Antes de instalar ou após um upgrade, verificar a integridade dos binários do Kubernetes garante que você está rodando o que o projeto publicou — e não algo adulterado.

---

## Por que fazer isso

Um binário comprometido na supply chain pode:
- Exfiltrar dados do cluster
- Abrir backdoors no apiserver
- Ignorar políticas de segurança silenciosamente

A verificação de hash é a linha de defesa mínima.

---

## Fluxo de verificação

```
1. Baixar o binário
2. Baixar o hash oficial publicado pelo projeto
3. Calcular o hash do binário baixado
4. Comparar — se diferente, não instalar
```

---

## Na prática — binários do Kubernetes

```bash
# Baixar o kubelet (exemplo versão 1.29.0)
curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubelet"

# Baixar o arquivo de hash oficial
curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubelet.sha256"

# Verificar
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
# kubelet: OK   ← deve aparecer isso

# Se adulterado:
# kubelet: FAILED
# sha256sum: WARNING: 1 computed checksum did NOT match
```

---

## sha512sum — versão mais forte

```bash
# Alguns projetos publicam SHA512
sha512sum kubelet > kubelet.sha512.local
diff kubelet.sha512.local kubelet.sha512.oficial

# Ou direto
echo "<hash-oficial>  kubelet" | sha512sum --check
```

---

## Verificar binários já instalados no cluster

No CKS, uma questão comum é: "um dos binários do node foi modificado — encontre e restaure."

```bash
# Hash dos binários instalados
sha256sum /usr/bin/kubelet
sha256sum /usr/bin/kubectl
sha256sum /usr/bin/kubeadm

# Comparar com o hash oficial da versão que está instalada
kubelet --version   # descobre a versão
# Vai no dl.k8s.io e compara
```

---

## Verificar imagens de container

Para imagens, o mecanismo é o digest SHA256 da camada de manifesto:

```bash
# Verificar digest de uma imagem
docker inspect --format='{{index .RepoDigests 0}}' k8s.gcr.io/kube-apiserver:v1.29.0

# Usar digest fixo no manifesto (em vez de tag)
# Tag pode ser sobrescrita — digest é imutável
image: k8s.gcr.io/kube-apiserver@sha256:abc123...
```

---

## No contexto de upgrade do cluster

Antes de cada upgrade:
```bash
# 1. Baixar os novos binários
# 2. Verificar hashes
# 3. Só então instalar
kubeadm upgrade apply v1.29.0

# kubeadm já verifica internamente, mas boa prática verificar manualmente também
```
