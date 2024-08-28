
# Tutorial Release Flow da Microsoft

## 1. Inicializar o Repositório

Comece inicializando o repositório Git e criando o branch principal.

```bash
git init
git checkout -b main
```

## 2. Desenvolvimento de Funcionalidades

Desenvolva novas funcionalidades criando branches de curta duração a partir do `main`.

```bash
git checkout -b func1 main
# Desenvolva a funcionalidade...
git commit -am "Implementa a funcionalidade 1"
git checkout main
git merge func1
git branch -d func1
```

## 3. Correção de Bugs

Para corrigir bugs, crie um branch a partir de `main`, faça a correção e mescle-o de volta.

```bash
git checkout -b bugfix1 main
# Corrige o bug...
git commit -am "Corrige o bug 1"
git checkout main
git merge bugfix1
git branch -d bugfix1
```

## 4. Preparar uma Versão de Release

Ao preparar uma nova versão, utilize o próprio branch `main`. Quando estiver pronto para o lançamento, crie uma tag.

```bash
# Certifique-se de que o main está pronto para o release
git tag v1.0  # Criação de tag simples
```

## 5. Deploy e Hotfixes

Se houver um problema em produção após o deploy, crie um branch de hotfix a partir de `main`, faça as correções necessárias e mescle de volta.

```bash
git checkout -b hotfix main
# Corrige o problema...
git commit -am "Corrige hotfix de produção"
git checkout main
git merge hotfix
git tag v1.0.1  # Criação de tag simples para o hotfix
git branch -d hotfix
```

## 6. Continuar com o Desenvolvimento

Após o release, continue o desenvolvimento de novas funcionalidades repetindo o processo.

```bash
git checkout -b func2 main
# Desenvolva a funcionalidade...
git commit -am "Implementa a funcionalidade 2"
git checkout main
git merge func2
git branch -d func2
```

---

### Resumo:

- **Branch Principal**: O `main` é o único branch longo.
- **Desenvolvimento**: Novas funcionalidades e correções de bugs são feitas em branches curtos que são mesclados de volta ao `main`.
- **Release**: Releases são realizados a partir do `main`, com criação de tags para identificar versões.
- **Hotfixes**: Correções rápidas são feitas diretamente a partir do `main` e também resultam em novas tags de versão.
