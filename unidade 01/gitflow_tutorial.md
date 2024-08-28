
# Tutorial GitFlow com Restrições

## 1. Inicializar o Repositório

Inicie o repositório Git e crie os branches principais:

```bash
git init
git checkout -b main
git checkout -b devel
```

## 2. Iniciar e Concluir uma Nova Funcionalidade

Crie um branch curto para a nova funcionalidade e, em seguida, mescle-o de volta ao branch `devel`.

```bash
git checkout -b feat1 devel
# Implementa a funcionalidade...
git commit -am "Implementa funcionalidade X"
git checkout devel
git merge feat1
git branch -d feat1
```

## 3. Preparar uma Versão de Release

Crie um branch de release a partir de `devel` para preparar a versão final.

```bash
git checkout -b rel1.0 devel
# Realiza ajustes finais e correções de bugs...
git commit -am "Preparação para versão 1.0"
```

## 4. Corrigir Defeitos no Release

Caso sejam encontrados defeitos durante a fase de release, corrija-os diretamente no branch de release.

```bash
git checkout rel1.0
# Corrige defeitos...
git commit -am "Corrige defeitos no release 1.0"
```

## 5. Concluir o Release e Mesclar com Main e Devel

Mescle as mudanças do release no branch `main` e sincronize `devel`.

```bash
git checkout main
git merge rel1.0
git tag v1.0  # Criação de tag simples, sem mensagens ou anotação
git checkout devel
git merge main
git branch -d rel1.0
```

## 6. Realizar uma Correção Rápida (Hotfix)

Se houver um problema crítico em produção, crie um branch de hotfix a partir de `main`.

```bash
git checkout -b hotfix main
# Corrige o erro crítico...
git commit -am "Corrige erro crítico em produção"
git checkout main
git merge hotfix
git tag v1.0.1  # Criação de tag simples para o hotfix
git checkout devel
git merge main
git branch -d hotfix
```

## 7. Continuar com o Desenvolvimento

Inicie uma nova funcionalidade ou continue com o desenvolvimento normal.

```bash
git checkout -b feat2 devel
# Implementa nova funcionalidade...
git commit -am "Implementa nova funcionalidade Y"
```

