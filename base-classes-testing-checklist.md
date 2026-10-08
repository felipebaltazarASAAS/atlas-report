# Checklist de Testes — Classes Base de ViewModel

> **Como usar este documento**: este checklist é comentado automaticamente no Pull Request pelo workflow `.github/workflows/base-classes-checklist.yml` sempre que o diff alterar uma das classes base listadas abaixo. Marque os itens no próprio comentário do PR (ou copie-os para a seção "Cenários testados") antes de solicitar o merge.

## Classes monitoradas

| Classe | Arquivo |
|---|---|
| `BaseViewModel` | `App/Shared/Asaas.Framework/Shared/Managers/Navigation/ViewModels/BaseViewModel.cs` |
| `BasePageIndexViewModel` | `App/Shared/Asaas.Framework/Shared/Managers/Navigation/ViewModels/BasePageIndexViewModel.cs` |

Essas classes são herdadas por praticamente todas as telas do app (`BaseViewModel`) e por todas as listagens paginadas (`BasePageIndexViewModel`). Uma regressão nelas não aparece na tela alterada no PR, e sim nas telas derivadas.

<!-- checklist:start -->
### Checklist obrigatório

- [ ] Validar funcionalidade de busca em pelo menos 3 telas que herdam da classe modificada
- [ ] Testar estados de loading/carregamento em telas derivadas
- [ ] Verificar se BindableProperties estão funcionando corretamente
- [ ] Confirmar que botões ficam habilitados após carregamento dos dados
- [ ] Testar em ambas as plataformas (Android e iOS)

Para alterações em `BaseViewModel`, incluir também ao menos uma tela que use `ViewModelProperties`/`BindableProperties` (ex.: Login).
<!-- checklist:end -->

## ViewModels sugeridas para validação

O bot não usa uma lista fixa de telas. A cada PR, o script `.github/scripts/base-classes-checklist.js` lê os arquivos `.cs` de `App/`, monta a árvore de herança e sorteia ViewModels concretas que herdam (direta ou indiretamente) da classe alterada:

| Classe alterada | Sorteio |
|---|---|
| `BaseViewModel` | 5 telas que herdam de `BasePageViewModel` e 1 popup que herda de `BasePopupViewModel` |
| `BasePageIndexViewModel` | 5 listagens que herdam de `BasePageIndexViewModel` |

O sorteio usa o número do PR como semente, então as sugestões são sempre as mesmas para o mesmo PR. Classes abstratas ficam de fora, e telas que deixaram de herdar da classe base (ex.: variantes `TestB` das listagens) nunca são sorteadas.

**Referência**: MB-3 (melhoria levantada em postmortem)
