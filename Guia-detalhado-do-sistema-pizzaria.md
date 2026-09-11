**Guia detalhado do seu sistema de pizzaria — leitura do código de 9 de setembro de 2026**

Este material explica o código que está escrito, o comportamento herdado do Django REST Framework e os pontos em que você o personalizou. Os exemplos marcados como didáticos servem para acompanhar a execução; não representam alterações realizadas nos arquivos.

Foram analisados os nove arquivos Python enviados e os dois arquivos de frontend anexados durante a conversa: `GerenciarIngredientes(1).tsx` e `GerenciarPizzas(1).tsx`. No texto, os componentes são chamados pelos seus nomes, `GerenciarIngredientes` e `GerenciarPizzas`. Existe uma tela antiga de ingredientes, de junho, mas ela não é a base desta explicação. Nos anexos Python, os arquivos sem `(1)` contêm o backend de ingredientes; os arquivos com `(1)` contêm o backend de pizzas. O `apps.py` recebido configura o app de pizzas.

Não foram fornecidos `settings.py`, o `urls.py` principal, `package.json`, o código de `Navbar`, os arquivos CSS nem as migrações. Portanto, autenticação global, configuração real do banco, montagem final das URLs, versões instaladas e detalhes internos de `Navbar` não podem ser confirmados por esses arquivos. As telas chamam explicitamente URLs iniciadas por `http://localhost:8000/api/`; esse é o endereço usado nos exemplos. A análise foi feita pela leitura do código, sem executar o projeto completo.

**1. O sistema funciona por camadas, e cada camada tem uma responsabilidade.**

Ao cadastrar uma pizza, o React mantém os valores digitados na memória do navegador. O Axios envia esses valores em uma requisição HTTP. O Django identifica a URL. O router do DRF associa o método HTTP a uma ação do ViewSet. A ação instancia um serializer, valida a entrada e solicita a gravação. O ORM do Django transforma operações com os models em comandos para o banco. A resposta volta pelo serializer, pelo HTTP e pelo Axios; finalmente, o React atualiza a tela.

| Parte | O que representa no seu projeto | Exemplo concreto |
|---|---|---|
| Componente React | Função que descreve a tela e organiza suas interações | `GerenciarPizzas()` |
| Estado React | Dados mantidos durante a utilização da tela | `novaPizza`, `pizzas` |
| Axios | Cliente que envia requisições HTTP | `axios.post(url, dados)` |
| URL | Endereço de uma operação da API | `/api/pizzas/` |
| Router | Registro que associa caminhos e métodos HTTP a ações | `DefaultRouter()` |
| ViewSet | Classe que atende às operações HTTP de um recurso | `PizzasViewSet` |
| Serializer | Define entrada, saída, conversão e validação dos dados da API | `PizzasSerializer` |
| Model | Classe que descreve dados persistentes e relações | `Pizzas` |
| ORM | Ferramentas do Django para consultar e gravar usando objetos | `Pizzas.objects.create(...)` |
| Banco | Local em que os registros persistem | Tabelas de pizzas e ingredientes |

API é a interface de comunicação do backend. JSON é o formato de dados utilizado nessas chamadas. HTTP é o protocolo que transporta as requisições e as respostas. Uma URL sozinha não significa cadastrar ou excluir: o método HTTP também participa dessa decisão.

Seu frontend não importa `PizzasViewSet` nem executa Python. Ele envia uma mensagem HTTP. Seu backend não chama `setPizzas`: ele devolve dados, e a função JavaScript registrada no `.then(...)` utiliza esses dados.

**2. Antes do CRUD, entenda classe, instância, parâmetro e chamada.**

```python
class Ingredientes(models.Model):
    nome = models.CharField(max_length=100, unique=True)
```

`class` declara uma classe. `Ingredientes` é o nome escolhido. `models.Model`, entre parênteses, é a classe base: essa é uma relação de herança. `Ingredientes` recebe mecanismos que o Django já implementou, como persistência, acesso aos campos e consulta por meio do manager.

Esta linha é diferente de criar um ingrediente:

```python
nome = models.CharField(max_length=100, unique=True)
```

Aqui você instancia um **objeto de configuração de campo**. Ele descreve o atributo `nome` para o Django. Não cadastra um ingrediente chamado `nome`. `max_length` e `unique` são argumentos nomeados passados à construção desse campo.

Agora compare:

```python
# Exemplo didático de instanciação, ainda sem INSERT:
ingrediente = Ingredientes(
    nome='Mussarela',
    quantidade_estoque=Decimal('10.000'),
    unidade_De_Medida='KG',
    custo_total=Decimal('300.00'),
    estoque_minimo=2,
    custoMedio=Decimal('30.00'),
)

# Persistência da instância:
ingrediente.save()
```

`Ingredientes(...)` cria um objeto Python na memória. `ingrediente` passa a referenciar esse objeto. `ingrediente.save()` pede ao Django que o grave. Para uma instância nova como a do exemplo, normalmente ocorre um `INSERT`. Para uma instância recuperada do banco e editada, normalmente ocorre um `UPDATE`.

Já `Ingredientes.objects.create(...)` reúne a construção de uma nova instância e sua gravação, devolvendo o objeto criado. Portanto, não é necessário acrescentar outro `.save()` apenas para efetivar essa criação.

```python
def perform_destroy(self, instance):
    instance.ativo = False
    instance.save(update_fields=['ativo'])
```

`def` declara uma função. Como está dentro da classe, essa função é um método. `self` é a instância da classe que está executando o método. `instance` é outro objeto recebido como argumento: neste caso, um ingrediente.

Nesse método, `self` é um `IngredientesViewSet`, enquanto `instance` é um `Ingredientes`. Os dois não são intercambiáveis.

Um **parâmetro** é um nome na definição, como `instance`. Um **argumento** é o valor fornecido na chamada. Em `self.perform_destroy(ingrediente)`, o objeto referenciado por `ingrediente` chega ao parâmetro `instance`. Python fornece `self` automaticamente quando o método é chamado por uma instância.

O `self` muda de significado conforme a classe:

| Dentro de | `self` representa |
|---|---|
| `Ingredientes.__str__` | Um ingrediente |
| `Pizzas.__str__` | Uma pizza |
| `IngredientesSerializer.validate` | Um serializer de ingredientes |
| `PizzasSerializer.update` | Um serializer de pizzas |
| `IngredientesViewSet.toggle_ativo` | O ViewSet que atende à requisição |

**3. Imports tornam nomes disponíveis; não executam o CRUD por si só.**

`from django.db import models` disponibiliza o módulo usado para declarar models e campos persistentes. O ponto em `models.CharField` significa acessar `CharField` dentro desse módulo.

`from rest_framework import serializers` disponibiliza o módulo de serializers do DRF. Quando você escreve `serializers.CharField(...)`, acessa uma classe desse módulo e cria um campo de API. A palavra `serializers` antes do ponto não é um objeto do seu ingrediente: é o nome do módulo importado.

`from .models import Ingredientes` importa uma classe do arquivo `models.py` do próprio pacote. O ponto inicial é um import relativo. Em `apps.ingredientes_estoque.models`, o caminho é explícito e percorre os pacotes do projeto.

`from decimal import Decimal` importa a classe de números decimais. `Decimal('0.00')` constrói um valor decimal a partir de texto, evitando introduzir a aproximação binária de um `float` como ponto de partida. Uma divisão ainda pode precisar de arredondamento: por exemplo, `1 / 3` não possui representação decimal finita.

`from rest_framework import viewsets` permite escrever `viewsets.ModelViewSet`. `from rest_framework.viewsets import ModelViewSet` permite usar diretamente `ModelViewSet`. As duas formas apontam à mesma classe base. No `views.py` de ingredientes, a importação direta de `ModelViewSet` está sobrando, pois a herança usa `viewsets.ModelViewSet`.

`ValidationError` representa uma falha esperada de validação. `Response` constrói a resposta do DRF. `status` fornece nomes para códigos HTTP, como `HTTP_200_OK`. `action` marca métodos adicionais para o router. `path` e `include` organizam URLs. `AppConfig` permite configurar um app Django.

O `Decimal` importado no model de ingredientes não é utilizado naquele arquivo. Já no model de pizzas ele participa do valor padrão de `custoProducao`. O comentário `# Create your models here.` não tem efeito na execução.

**4. O model de ingredientes define os dados de cada ingrediente.**

```python
class Ingredientes(models.Model):
    class UnidadeDeMedida(models.TextChoices):
        KILOGRAMA = 'KG', 'Kilogramas'
```

`UnidadeDeMedida` é uma enumeração de opções textuais, declarada dentro de `Ingredientes` para organizar uma configuração relacionada a essa classe. `'KG'` é o valor armazenado; `'Kilogramas'` é o rótulo legível. `UnidadeDeMedida.choices` fornece as opções para o campo. No código atual, somente `KG` está definido.

| Campo | Declaração | Consequência |
|---|---|---|
| `nome` | `CharField(max_length=100, unique=True)` | Texto de até 100 caracteres e unicidade |
| `quantidade_estoque` | `DecimalField(max_digits=10, decimal_places=3)` | Quantidade com até 10 dígitos totais, sendo 3 decimais |
| `unidade_De_Medida` | `CharField(max_length=10, choices=...)` | Código da unidade dentre as opções |
| `custo_total` | `DecimalField(10, 2)` | Valor total monetário armazenado |
| `estoque_minimo` | `FloatField()` | Limite numérico para o estoque |
| `custoMedio` | `DecimalField(10, 2)` | Custo médio armazenado por unidade |
| `ativo` | `BooleanField(default=True)` | Situação inicial ativa |

`max_digits=10` conta todos os dígitos, não apenas a parte inteira. Com três casas decimais, ficam até sete posições inteiras. Assim, `1234567.890` ocupa os dez dígitos. Com duas casas, ficam até oito posições inteiras.

Você não escreveu `null=True` nesses campos do model. Portanto, eles não aceitam `NULL` no banco por essa declaração. Isso é diferente de aceitar a string vazia `''` em uma tela.

`ativo=False` não apaga o registro. Apenas muda um atributo. Não esconde automaticamente o ingrediente das consultas: isso depende do filtro escrito na consulta ou na tela.

O campo `id` pode ser acrescentado automaticamente pelo Django quando não existe outra chave primária declarada. Para o app de pizzas, o `apps.py` recebido define `BigAutoField`. A configuração específica do app de ingredientes não foi enviada.

```python
def __str__(self):
    return self.nome
```

`__str__` é um método especial de representação textual. Quando um objeto precisa ser mostrado como texto, pode aparecer seu nome em vez de uma representação genérica. Isso ajuda no shell e no admin; não muda o nome da tabela nem decide a estrutura completa do JSON.

O model tem `estoque_minimo`, mas esse campo sozinho não implementa uma rotina de alertas. Também não existe, nesses arquivos, uma entidade separada responsável pelo saldo atual: o saldo está no próprio `Ingredientes.quantidade_estoque`.

Models são a declaração da estrutura; migrações aplicam mudanças de estrutura no banco. Alterar um campo normalmente exige `makemigrations` e `migrate`. Alterar somente uma função React ou uma validação do serializer normalmente não exige uma migração. [Documentação de models do Django](https://docs.djangoproject.com/en/5.2/topics/db/models/).

**5. Pizza e ingrediente se relacionam pela composição.**

O model `Pizzas` declara `nome`, `precoVenda`, `tamanho`, `custoProducao`, `ativo` e a associação `ingredientes`.

`Tamanho` funciona como `UnidadeDeMedida`, mas com os códigos `P`, `M` e `G`. O campo `tamanho` tem `max_length=1` porque o código armazenado possui uma letra. O rótulo completo não precisa caber nessa coluna.

`precoVenda` e `custoProducao` são valores decimais com duas casas. `custoProducao` tem valor padrão `Decimal('0.00')`. Isso não é uma fórmula automática no banco: o cálculo efetivo aparece no serializer e na view.

```python
ingredientes = models.ManyToManyField(
    Ingredientes,
    through='PizzaIngrediente',
    related_name='pizzas',
)
```

O primeiro argumento informa a classe relacionada. `ManyToManyField` representa muitos para muitos: uma pizza pode utilizar vários ingredientes, e um ingrediente pode participar de várias pizzas.

`through='PizzaIngrediente'` determina que a associação será armazenada usando seu model intermediário. A string permite referenciar a classe antes de sua declaração posterior no arquivo.

`related_name='pizzas'` define o caminho de acesso no sentido inverso. É por isso que, partindo de um ingrediente, funciona `ingrediente.pizzas.all()`.

```python
class PizzaIngrediente(models.Model):
    pizza = models.ForeignKey(
        Pizzas,
        on_delete=models.CASCADE,
        related_name='composicao',
    )
    ingrediente = models.ForeignKey(
        Ingredientes,
        on_delete=models.CASCADE,
        related_name='composicoes_pizza',
    )
    quantidade = models.DecimalField(max_digits=10, decimal_places=3)
```

Cada `PizzaIngrediente` é uma linha da receita: uma pizza, um ingrediente e uma quantidade. A quantidade pertence à associação porque a mesma mussarela pode ser usada em quantidades diferentes em pizzas diferentes.

Exemplo didático:

| Pizza | Ingrediente | Quantidade na receita |
|---|---|---:|
| Mussarela grande | Massa | 0,300 KG |
| Mussarela grande | Mussarela | 0,200 KG |
| Calabresa grande | Mussarela | 0,100 KG |

O estoque da mussarela continua sendo um só, em `Ingredientes.quantidade_estoque`. As quantidades `0,200` e `0,100` descrevem receitas. Cadastrar uma receita não reduz o estoque no seu código atual.

As duas `ForeignKey` criam referências a registros de outras tabelas. No uso de objetos, `item.ingrediente` devolve o objeto relacionado; `item.ingrediente_id` fornece a chave estrangeira. Conceitualmente, o banco armazena os identificadores, não uma cópia inteira da classe Python.

| Expressão | Tipo de resultado |
|---|---|
| `pizza.ingredientes.all()` | Ingredientes relacionados à pizza |
| `pizza.composicao.all()` | Linhas `PizzaIngrediente`, incluindo a quantidade |
| `ingrediente.pizzas.all()` | Pizzas relacionadas ao ingrediente |
| `ingrediente.composicoes_pizza.all()` | Linhas da composição em que o ingrediente aparece |
| `item.pizza` | A pizza daquela linha |
| `item.ingrediente` | O ingrediente daquela linha |

Esses acessos de coleção são managers de relações. Você pode construir consultas sobre eles, como `.filter(ativo=True)` e `.exists()`.

`on_delete=models.CASCADE` participa da exclusão física feita pelo ORM. Ao excluir uma pizza, as linhas intermediárias que a referenciam são excluídas; os ingredientes continuam existindo. Ao excluir fisicamente um ingrediente por outro caminho, o `CASCADE` atual também permite excluir suas linhas intermediárias. Desativar com `ativo=False` não aciona cascata.

O impedimento de excluir ingrediente vinculado está na view, não nesse `on_delete`. Portanto, não é uma proteção global que automaticamente alcança qualquer script ou operação administrativa.

Seu código não define uma restrição de unicidade para `(pizza, ingrediente)`. O frontend impede repetições na composição, mas a API ainda pode receber repetições se alguém chamá-la diretamente. A obrigação de haver ao menos um ingrediente é validada pelo serializer da pizza; o relacionamento sozinho não impõe esse mínimo no banco.

O `__str__` da associação utiliza uma f-string: `f"{self.pizza.nome} - {self.ingrediente.nome}"`. As expressões entre chaves são avaliadas e inseridas no texto.

**6. O `apps.py` configura o app; não instancia uma pizza.**

```python
class PizzasConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.pizzas'
```

`PizzasConfig` herda da classe de configuração de apps. `name` aponta ao pacote Python que contém o app. `default_auto_field` define o tipo padrão das chaves primárias automáticas desse app.

O Django carrega os apps registrados em `INSTALLED_APPS`, no `settings.py`. Como esse arquivo não foi fornecido, aqui explicamos o significado da configuração recebida, sem afirmar qual linha exata está no seu `INSTALLED_APPS`.

Sem personalização de `db_table`, o Django normalmente combina o rótulo do app e o nome do model em minúsculas para nomear a tabela. Assim, o app `pizzas` e o model `Pizzas` resultam normalmente em `pizzas_pizzas`. O plural da classe não impede o funcionamento, embora cada instância continue representando uma pizza.

**7. Serializer tem entrada, validação, gravação e saída — operações diferentes.**

`IngredientesSerializer(serializers.ModelSerializer)` herda de `ModelSerializer`. A classe base consegue derivar campos e parte das validações a partir do model indicado em `Meta`. Também oferece implementações padrão de `create` e `update` para casos simples.

Uma forma precisa de acompanhar seu uso é esta:

| Expressão | O que acontece |
|---|---|
| `IngredientesSerializer(data=dados)` | Cria um serializer para receber dados de entrada |
| `serializer.is_valid()` | Converte e valida a entrada; não salva |
| `serializer.validated_data` | Dados internos que passaram pela validação |
| `serializer.errors` | Erros encontrados na validação |
| `serializer.save()` | Chama `create` ou `update`, conforme exista uma instância |
| `IngredientesSerializer(ingrediente)` | Prepara a representação de um objeto existente |
| `serializer.data` | Obtém a representação de saída |

Na edição, a construção tem dois elementos importantes:

```python
serializer = IngredientesSerializer(
    instance=ingrediente_existente,
    data=dados_recebidos,
    partial=True,
)
```

`instance` identifica o objeto que será alterado. `data` contém o que veio do cliente. `partial=True` permite que campos obrigatórios sejam omitidos nessa operação parcial. Isso não corrige automaticamente métodos personalizados que assumam a presença de todos os campos.

Para representar uma coleção, usa-se `many=True`. O DRF utiliza um `ListSerializer` envolvendo o serializer de cada elemento. Não significa criar várias linhas automaticamente ao ler uma lista.

`serializer_class = IngredientesSerializer` guarda uma referência à **classe**, sem instanciá-la ali. Mais tarde, `self.get_serializer(...)` consulta essa configuração e constrói a **instância** do serializer com os argumentos adequados. Por padrão, também inclui contexto, como a requisição e a view. [Documentação de serializers do DRF](https://www.django-rest-framework.org/api-guide/serializers/).

**8. O serializer de ingredientes define quais campos entram e quais saem.**

```python
class Meta:
    model = Ingredientes
    fields = [
        'id', 'nome', 'quantidade_estoque', 'unidade_De_Medida',
        'custo_total', 'estoque_minimo', 'custoMedio', 'ativo',
    ]
    read_only_fields = ['custoMedio', 'ativo']
```

`Meta` é uma classe interna de configuração reconhecida pelo `ModelSerializer`. `model` informa de qual model derivar informações. `fields` lista os campos expostos por esse serializer. `read_only_fields` determina campos que podem aparecer na resposta, mas não são editáveis por essa entrada normal do serializer.

Consequentemente, o cliente não controla `custoMedio` enviando-o no JSON comum. A view calcula esse valor e o passa a `serializer.save(custoMedio=...)`. `ativo` é alterado pela ação `toggle_ativo` ou pelo `perform_destroy` específico, que manipulam o model diretamente.

Um campo somente de leitura fornecido na entrada normalmente é ignorado pelo serializer; não se deve presumir que seu envio causará um erro explícito. A configuração vale para esse serializer, não impede que código Python autorizado faça `ingrediente.ativo = False`.

O `id` automático é tratado como somente leitura pelo `ModelSerializer`, mesmo sem constar nessa lista manual.

```python
custo_total = serializers.DecimalField(
    max_digits=10,
    decimal_places=2,
    required=False,
    allow_null=True,
)
```

Aqui você declara explicitamente o campo da API e substitui a configuração que seria derivada automaticamente do model. `required=False` permite que a chave não seja enviada. `allow_null=True` permite que ela venha com JSON `null`, que se torna `None` em Python. São situações diferentes.

| Entrada | Significado |
|---|---|
| Sem a chave `custo_total` | Campo omitido |
| `"custo_total": null` | Campo enviado com valor nulo |
| `"custo_total": ""` | Campo enviado como texto vazio |
| `"custo_total": "0.00"` | Valor decimal zero |

Permitir nulo no serializer não torna a coluna do banco anulável. Seu `validate` tenta normalizar o cadastro sem estoque para custo zero; existem lacunas na atualização, detalhadas adiante. Texto vazio em entrada JSON não é automaticamente um decimal válido: pode ser rejeitado pelo campo antes de chegar ao `validate` geral. [Opções dos campos do DRF](https://www.django-rest-framework.org/api-guide/fields/).

**9. As validações de ingredientes são chamadas pelo DRF durante `is_valid()`.**

Você não precisa chamar `validate_nome(...)` manualmente no ViewSet. O nome tem significado para o framework: `validate_` seguido do nome do campo identifica sua validação personalizada. Esses métodos são ganchos reconhecidos pelo DRF; não é necessário existir uma implementação específica `validate_nome` na classe base.

A sequência relevante é: interpretar os campos editáveis; converter seus tipos; aplicar validações dos campos, incluindo validadores automáticos; executar os métodos `validate_<campo>` correspondentes; se os campos forem válidos, aplicar validações do conjunto e `validate(attrs)`.

Se um campo obrigatório estiver faltando ou não puder ser convertido, pode haver erro antes de sua validação personalizada. A validação geral não é uma rotina que sempre roda apesar de todos os erros anteriores. Campos omitidos em uma atualização parcial também não passam por sua validação individual naquele envio.

`validate_nome(self, value)` recebe o texto do nome. `value.strip()` remove espaços do início e do fim. A consulta `Ingredientes.objects.filter(nome__iexact=nome)` procura nomes iguais sem distinguir maiúsculas de minúsculas.

O `__` em `nome__iexact` faz parte da sintaxe de consultas do ORM: `nome` é o campo e `iexact` é a operação de comparação. `.filter(...)` cria um `QuerySet`; `.exists()` verifica se há pelo menos um registro correspondente.

```python
if self.instance:
    ingredientes = ingredientes.exclude(id=self.instance.id)
```

Na criação, o serializer não recebeu um ingrediente existente. Na edição, recebeu. Excluir o próprio ID da consulta evita que um ingrediente seja considerado duplicado de si mesmo quando mantém o nome.

Se a consulta encontrar outro registro, `raise serializers.ValidationError(...)` interrompe a validação desse campo e fornece uma mensagem de erro. Se tudo estiver certo, `return nome` devolve o valor que será usado. O retorno importa: a validação também pode normalizar os dados.

`unique=True` no model normalmente gera um validador automático de unicidade no serializer. Por isso, uma duplicidade exata pode ser rejeitada antes de chegar à sua mensagem manual. Sua busca com `iexact` acrescenta explicitamente a comparação sem distinção de caixa. Uma consulta prévia sozinha não resolve todas as disputas entre duas gravações simultâneas; para garantir unicidade sem distinção de caixa em todos os caminhos, a regra também precisa de proteção apropriada no banco. [Validadores do DRF](https://www.django-rest-framework.org/api-guide/validators/).

As outras validações individuais são diretas:

| Método | Regra atual |
|---|---|
| `validate_quantidade_estoque` | Rejeita quantidade menor que zero |
| `validate_custo_total` | Aceita `None` nessa etapa; rejeita custo negativo |
| `validate_estoque_minimo` | Rejeita mínimo negativo |

Zero é permitido nesses três campos. No custo total, existir um valor e o valor ser maior que zero são regras diferentes: seu backend exige informação de custo quando há estoque, mas permite custo igual a zero.

`validate(self, attrs)` recebe o dicionário dos valores internos já validados individualmente. O nome `attrs` é uma convenção local; poderia ter outro nome sem mudar o mecanismo.

```python
quantidade = attrs.get('quantidade_estoque')
custo_total = attrs.get('custo_total')
```

`.get(...)` devolve o valor ou `None` quando a chave não existe. Já `attrs['campo']` gera `KeyError` se a chave estiver ausente.

O comportamento do seu método é:

1. Se quantidade não veio, retorna `attrs` imediatamente.
2. Converte a quantidade para `Decimal` passando antes por `str`; normalmente ela já veio convertida pelo campo decimal.
3. Se quantidade é positiva e custo está ausente/nulo, levanta erro associado a `custo_total`.
4. Se quantidade é zero, custo está ausente/nulo e não existe `self.instance`, define `custo_total = Decimal('0.00')`.
5. Retorna o dicionário, eventualmente normalizado.

O `and not self.instance` limita essa normalização ao cadastro. É por isso que cadastro e atualização com zero e custo omitido não seguem exatamente o mesmo caminho.

**10. O serializer da composição transforma IDs em objetos relacionados.**

```python
ingrediente_id = serializers.PrimaryKeyRelatedField(
    queryset=Ingredientes.objects.filter(ativo=True),
    source='ingrediente',
)
```

`PrimaryKeyRelatedField` é um campo de relacionamento. Na entrada, recebe uma chave primária, procura um objeto válido no `queryset` e entrega esse objeto ao restante da validação.

Exemplo de entrada:

```json
{"ingrediente_id": 7, "quantidade": "0.200"}
```

Representação didática após validar:

```python
{
    'ingrediente': objeto_Ingredientes_com_id_7,
    'quantidade': Decimal('0.200'),
}
```

`source='ingrediente'` explica a mudança de nome. O nome público é `ingrediente_id`; o atributo interno do model é `ingrediente`. Você não precisa fazer uma segunda busca manual só para converter esse ID na hora de criar a associação: o campo já encontrou o objeto.

O filtro `ativo=True` limita os objetos aceitos na **entrada**. Ele não apaga associações existentes e não esconde automaticamente ingredientes inativos na **saída** de uma pizza. A saída pode continuar exibindo uma associação antiga, inclusive seu `ingrediente_ativo=False`. [Relacionamentos nos serializers](https://www.django-rest-framework.org/api-guide/relations/).

No seu arquivo atual, o filtro também rejeita um ingrediente inativo que já pertença à pizza quando a edição envia novamente a composição inteira. A distinção entre “novo ingrediente inativo” e “ingrediente inativo já vinculado” ainda não está implementada nessa versão.

| Campo público | `source` | Explicação |
|---|---|---|
| `ingrediente_id` | `ingrediente` | Referência pelo ID; aceita entrada válida |
| `ingrediente_nome` | `ingrediente.nome` | Lê o nome do objeto relacionado |
| `unidade_De_Medida` | `ingrediente.unidade_De_Medida` | Lê a unidade do ingrediente |
| `ingrediente_ativo` | `ingrediente.ativo` | Lê se o ingrediente está ativo |
| `custo_medio` | `ingrediente.custoMedio` | Lê o custo médio do ingrediente |
| `quantidade` | Nome do próprio campo | Quantidade daquela linha da receita |

Os quatro campos descritivos são `read_only=True`. Eles não criam colunas duplicadas em `PizzaIngrediente`, nem permitem renomear um ingrediente ao editar a pizza. Seus valores vêm da relação.

`validate_quantidade` rejeita `value <= 0`. Aqui zero não é aceito: uma linha presente na receita precisa ter quantidade positiva.

O import `from .models import Ingredientes` no serializer de pizzas funciona porque o `models.py` de pizzas importou esse nome e o disponibilizou no módulo. Não é um segundo model de ingredientes.

**11. O serializer de pizzas apresenta nomes de API diferentes dos nomes do model.**

```python
preco_de_venda = serializers.DecimalField(
    source='precoVenda',
    max_digits=10,
    decimal_places=2,
)

ingredientes = PizzaIngredienteSerializer(
    source='composicao',
    many=True,
)
```

O JSON usa `preco_de_venda`, enquanto o model usa `precoVenda`. Na entrada, `source` direciona o valor para a chave interna correta. Na saída, usa o atributo interno para produzir a chave pública.

O campo público `ingredientes` utiliza o serializer da **composição**. Seu `source='composicao'` aponta para o `related_name` da chave estrangeira `PizzaIngrediente.pizza`. Assim, o serializer enxerga as linhas com suas quantidades, não apenas uma lista simples de ingredientes.

Exemplo didático de JSON recebido:

```json
{
  "nome": "Mussarela grande",
  "preco_de_venda": "45.00",
  "tamanho": "G",
  "ingredientes": [
    {"ingrediente_id": 7, "quantidade": "0.200"},
    {"ingrediente_id": 8, "quantidade": "0.300"}
  ]
}
```

Estrutura interna aproximada depois da validação:

```python
{
    'nome': 'Mussarela grande',
    'precoVenda': Decimal('45.00'),
    'tamanho': 'G',
    'composicao': [
        {'ingrediente': ingrediente_7, 'quantidade': Decimal('0.200')},
        {'ingrediente': ingrediente_8, 'quantidade': Decimal('0.300')},
    ],
}
```

Repare nas duas transformações: `preco_de_venda` vira `precoVenda`, e `ingredientes` vira `composicao`. Dentro de cada item, `ingrediente_id` vira `ingrediente`, com o objeto encontrado.

`validate_preco_de_venda` continua usando o **nome público do campo do serializer**, apesar de `source` ser `precoVenda`. Da mesma maneira, o método é `validate_ingredientes`, não `validate_composicao`.

`validate_nome` repete a lógica de remover espaços, procurar duplicidades sem distinguir caixa e excluir a própria pizza na edição. Como a unicidade atual considera somente o nome, duas pizzas com o mesmo nome e tamanhos diferentes continuam sendo rejeitadas. Não existe nesse código a regra alternativa de unicidade por `(nome, tamanho)`.

`validate_ingredientes` rejeita a lista vazia. `validate_preco_de_venda` exige valor maior que zero. `custoProducao` e `ativo` são somente leitura na entrada normal.

**12. O CRUD existe por herança, mesmo quando você não escreveu todas as funções.**

`ModelViewSet` reúne comportamentos de criação, listagem, consulta individual, atualização e exclusão. Esses comportamentos vêm de classes reutilizáveis chamadas mixins. No seu caso, os métodos superiores continuam herdados e chamam alguns métodos que você sobrescreveu. [ViewSets do DRF](https://www.django-rest-framework.org/api-guide/viewsets/).

| HTTP | Endereço relativo | Ação do ViewSet | Operação |
|---|---|---|---|
| `POST` | `/ingredientes/` | `create` | Criar ingrediente |
| `GET` | `/ingredientes/` | `list` | Listar ingredientes |
| `GET` | `/ingredientes/7/` | `retrieve` | Consultar um ingrediente |
| `PUT` | `/ingredientes/7/` | `update` | Atualizar com validação não parcial |
| `PATCH` | `/ingredientes/7/` | `partial_update` | Atualizar parcialmente |
| `DELETE` | `/ingredientes/7/` | `destroy` | Executar exclusão configurada |

As mesmas ações são disponibilizadas para `/pizzas/` e `/pizzas/7/`.

Uma diferença central é que existem métodos chamados `create` em classes diferentes:

| Método | Papel |
|---|---|
| `ViewSet.create(request, ...)` | Organiza a operação HTTP de cadastro |
| `ViewSet.perform_create(serializer)` | Gancho chamado durante esse cadastro |
| `Serializer.create(validated_data)` | Cria o objeto a partir dos dados validados |
| `Model.objects.create(**dados)` | Constrói e grava o registro pelo ORM |

O mesmo vale para `update`: a ação HTTP e o método do serializer não são a mesma função, embora compartilhem o nome.

Sobrescrever significa declarar, na classe filha, um método com o nome de um método herdado, substituindo sua implementação para aquela classe. Quando o método herdado chama `self.perform_create(...)`, Python encontra sua implementação personalizada.

Você não sobrescreveu `create` nas views. Sobrescreveu `perform_create`. Portanto, a ação herdada ainda instancia o serializer, chama a validação e monta a resposta de cadastro; somente a etapa de persistência foi personalizada.

Em `PizzasSerializer`, você realmente sobrescreveu os métodos `create` e `update` do serializer porque a composição aninhada precisa ser gravada explicitamente. Em `IngredientesSerializer`, esses dois métodos continuam herdados.

**13. As URLs conectam um endereço ao ViewSet e ao método HTTP.**

```python
router = DefaultRouter()
router.register(
    r'ingredientes',
    IngredientesViewSet,
    basename='ingredientes',
)

urlpatterns = [
    path('', include(router.urls)),
]
```

`DefaultRouter()` instancia o router. `router` é a variável que referencia esse objeto. `register` recebe o prefixo da rota, a classe do ViewSet e um nome-base para os nomes das rotas.

Você passa `IngredientesViewSet`, sem parênteses, porque registra a classe. Não cria manualmente uma instância para reutilizá-la entre todos os usuários.

O prefixo `r` em `r'ingredientes'` indica uma string bruta de Python. Como esse texto não contém escapes com barra invertida, seu valor é o mesmo de `'ingredientes'`. Esse `r` não significa rota nem requisição.

`basename='ingredientes'` participa de nomes internos como `ingredientes-list` e `ingredientes-detail`. Ele não acrescenta `/api/` nem substitui o primeiro argumento. Como suas views fornecem `get_queryset()` em vez de um atributo de classe `queryset`, explicitar `basename` evita depender de uma inferência que o router não consegue fazer a partir de um `queryset` declarado.

`router.urls` fornece os padrões gerados. `include(...)` delega a resolução para esse conjunto. `path('', ...)` não adiciona outro prefixo nesse nível. `urlpatterns` é a lista de rotas que o Django procura no módulo.

Os arquivos recebidos definem os prefixos `ingredientes` e `pizzas`. Para chegar a `/api/ingredientes/`, algum arquivo acima precisa incluí-los sob `api/`. Uma montagem possível seria `path('api/', include(...))`, mas a linha real não pode ser apontada porque o arquivo principal não foi enviado. [Resolução de URLs do Django](https://docs.djangoproject.com/en/5.2/topics/http/urls/).

O router gera, conceitualmente, ligações como `GET` para `list` e `POST` para `create` no endereço de coleção. No endereço com identificador, liga `GET` a `retrieve`, `PUT` a `update`, `PATCH` a `partial_update` e `DELETE` a `destroy`.

`pk` significa chave primária. Na rota `/pizzas/7/`, o valor capturado identifica a pizza procurada. Por padrão, `get_object()` usa a chave primária sobre a consulta da view. Se não encontrar, devolve erro HTTP 404; também participa da verificação de permissões do objeto configuradas no projeto.

| Ação adicional | Rota relativa gerada | Método |
|---|---|---|
| Ingrediente: `toggle_ativo` | `/ingredientes/{pk}/toggle_ativo/` | `POST` |
| Ingrediente: `adicionar_estoque` | `/ingredientes/{pk}/adicionar_estoque/` | `POST` |
| Pizza: `toggle_ativo` | `/pizzas/{pk}/toggle_ativo/` | `POST` |

O `@action(detail=True, methods=['post'])` é o que faz essas funções extras entrarem nas rotas do router. `detail=True` inclui um objeto identificado por `pk`; `detail=False` serviria para uma operação da coleção. Uma função comum como `calcular_custo_producao` não vira endpoint só porque está dentro do ViewSet. [Routers do DRF](https://www.django-rest-framework.org/api-guide/routers/).

**14. Uma requisição instancia a view, e a ação cria o serializer apropriado.**

No carregamento do projeto, Python define as classes e os routers registram as rotas. Nesse momento, a declaração do model não significa cadastrar registros.

Quando chega uma requisição, a resolução de URLs encontra a função produzida para o ViewSet. Esse mecanismo constrói uma instância da view para atender à chamada e associa o verbo HTTP à ação correspondente. O DRF fornece o objeto `request`, com dados, método, usuário e outros elementos.

`request.data` fornece os dados interpretados por parsers do DRF, conforme o tipo de conteúdo. Para JSON, transforma o corpo da requisição em uma estrutura utilizável em Python. O parser interpretar JSON e o serializer validar os valores são etapas diferentes. [Requests do DRF](https://www.django-rest-framework.org/api-guide/requests/).

No cadastro, a ação passa `data=request.data` ao serializer. Na edição, também passa a instância recuperada. Na listagem, passa uma coleção e `many=True`. Na consulta individual e nas respostas das suas ações, passa o objeto a representar.

Quando o backend devolve `Response(serializer.data)`, ainda há o processamento de resposta do DRF, que seleciona a representação adequada. No fluxo JSON das suas telas, os dados são renderizados como JSON. O Axios recebe essa resposta, interpreta seu conteúdo e disponibiliza o objeto em `response.data`.

`serializer.data` no Python e `response.data` no JavaScript não são a mesma variável nem ocupam a mesma memória. O HTTP transmite uma representação entre os processos.

**15. CREATE de ingredientes, acompanhando cada chamada.**

Considere o JSON didático:

```json
{
  "nome": "Mussarela",
  "quantidade_estoque": "10.000",
  "unidade_De_Medida": "KG",
  "custo_total": "300.00",
  "estoque_minimo": 2
}
```

1. O frontend chama `axios.post('http://localhost:8000/api/ingredientes/', ingredienteParaEnviar)`.
2. A URL é resolvida e o router seleciona `IngredientesViewSet.create` para esse `POST`.
3. `create` é herdado. Ele obtém uma instância de `IngredientesSerializer` com os dados recebidos.
4. A ação chama `serializer.is_valid(raise_exception=True)`.
5. Os campos convertem valores e executam as validações descritas. Falha gera resposta 400, sem chegar à etapa normal de salvar.
6. A ação herdada chama `self.perform_create(serializer)`.
7. Python executa o seu `IngredientesViewSet.perform_create`.
8. Seu método lê `quantidade_estoque` e `custo_total` em `serializer.validated_data`.
9. Calcula `300 / 10 = 30`.
10. Chama `serializer.save(custoMedio=custo_medio)`.
11. O serializer não tem instância anterior, então usa seu `create` herdado.
12. A criação padrão usa o model para gravar o novo ingrediente. O campo `ativo` recebe o padrão `True`; o ID automático é atribuído.
13. O objeto criado passa a ser a instância mantida pelo serializer.
14. A ação herdada usa a representação do serializer e devolve, no fluxo normal, HTTP 201.
15. O `.then(...)` do frontend recebe os dados salvos e acrescenta esse ingrediente ao estado da tabela.

Este é o seu gancho:

```python
def perform_create(self, serializer):
    quantidade = serializer.validated_data['quantidade_estoque']
    custo_total = serializer.validated_data['custo_total']

    if quantidade > 0:
        custo_medio = custo_total / quantidade
    else:
        custo_medio = Decimal('0.00')

    serializer.save(custoMedio=custo_medio)
```

`serializer` é a instância já validada fornecida pela ação. Os colchetes pressupõem a existência das chaves. Na criação com estoque zero e custo omitido/nulo, seu `validate` preenche o custo com zero antes desse ponto.

O argumento `custoMedio=custo_medio` acrescenta um valor decidido no servidor à gravação. O lado esquerdo é o nome do atributo do model; o direito é a variável local que você calculou.

O `else` evita divisão por zero e escolhe custo médio zero. Isso não consulta histórico: é um cálculo usando os valores dessa operação.

**16. READ de ingredientes: listar e consultar um registro.**

```python
def get_queryset(self):
    return Ingredientes.objects.all().order_by('id')
```

`objects` é o manager do model. `.all()` inicia uma consulta de todos os ingredientes. `.order_by('id')` ordena pelo ID em ordem crescente. O resultado é um `QuerySet`, que normalmente adia a execução da consulta até os resultados serem necessários.

Para `GET /api/ingredientes/`, a ação `list` obtém essa consulta, aplica mecanismos de filtro e paginação eventualmente configurados, serializa a coleção e devolve os dados.

Para `GET /api/ingredientes/7/`, a ação `retrieve` chama `get_object()`, que parte da consulta da view e procura o identificador. Depois, serializa um objeto.

Em uma leitura, não é necessário executar `is_valid()` para simplesmente representar um objeto já existente. Não existe entrada de cadastro a validar nesse passo.

Seu `get_queryset()` não filtra por `ativo=True`. Assim, ativos e inativos estão disponíveis. Isso explica por que desativar não remove automaticamente a linha da tabela.

As telas esperam uma lista diretamente em `response.data`. Se o `settings.py` habilitar paginação que devolva um objeto com `results`, esse contrato exigirá ajuste no frontend. Como a configuração não foi enviada, não é possível afirmar qual paginação está ativa.

**17. UPDATE de ingredientes: instância existente e dados novos têm papéis diferentes.**

Na edição inline, a tela chama `axios.put('/api/ingredientes/7/', ingredienteParaEnviar)`.

1. O router seleciona `update`, herdado do DRF.
2. A ação usa `get_object()` para recuperar o ingrediente 7.
3. Instancia o serializer com esse objeto e com os novos dados.
4. Chama `is_valid(raise_exception=True)`.
5. `self.instance` dentro do serializer agora representa o ingrediente 7. Por isso a validação do nome exclui o próprio ID da busca de duplicidade.
6. A ação chama seu `perform_update(serializer)`.
7. Você recalcula o custo médio e chama `serializer.save(custoMedio=...)`.
8. Como o serializer tem instância existente, ele escolhe `update`, e não `create`.
9. O `update` herdado do serializer atribui os valores validados à instância e salva.
10. A ação HTTP devolve os dados atualizados, normalmente com status 200.
11. A tela substitui somente a linha correspondente ao mesmo ID em seu array.

```python
def perform_update(self, serializer):
    quantidade = serializer.validated_data['quantidade_estoque']
    custo_total = serializer.validated_data['custo_total']
    custo_medio = (
        custo_total / quantidade
        if quantidade > 0
        else Decimal('0.00')
    )
    serializer.save(custoMedio=custo_medio)
```

A expressão `A if condição else B` escolhe um valor em Python: divide quando a quantidade é positiva; usa zero no outro caso.

Editar o saldo por `PUT` substitui os valores pelos enviados. Isso difere de `adicionar_estoque`, que soma uma entrada aos valores existentes.

`PATCH` também é disponibilizado pelo `ModelViewSet`. O `partial_update` herdado encaminha a operação à atualização com `partial=True`. Entretanto, seu `perform_update` acessa as duas chaves com colchetes. Um `PATCH` contendo apenas `{"nome": "Mussarela especial"}` passa pela ideia de atualização parcial, mas esse gancho tenta ler uma quantidade ausente e pode gerar `KeyError`.

Além disso, um envio com quantidade zero e custo omitido pode deixar a chave ausente na atualização. Se custo vier `null`, seu serializer pode mantê-lo como `None`, mas o model não permite nulo. Esses casos mostram que a rota existir e seu código personalizado tratar corretamente todos os seus usos são questões distintas.

`PUT` utiliza validação não parcial, exigindo os campos obrigatórios desse serializer. Não significa que todos os campos somente de leitura precisam ser enviados, nem que todos os campos opcionais omitidos serão necessariamente apagados.

**18. DELETE de ingredientes foi personalizado para desativar.**

```python
def perform_destroy(self, instance):
    if instance.pizzas.exists():
        raise ValidationError(
            'Não é possível deletar este ingrediente, pois ele está associado a uma pizza.'
        )
    instance.ativo = False
    instance.save(update_fields=['ativo'])
```

O fluxo é: `DELETE` seleciona `destroy`; `destroy` recupera o objeto com `get_object`; chama `perform_destroy(instance)`; seu método verifica as relações e, se permitido, salva `ativo=False`; a ação herdada normalmente responde 204, sem corpo.

Você substituiu a parte que normalmente faria `instance.delete()`. Portanto, o registro permanece no banco.

`instance.pizzas.exists()` considera qualquer pizza relacionada, inclusive inativa. Caso encontre uma, levanta um `ValidationError`, que no tratamento padrão do DRF corresponde a resposta 400.

`update_fields=['ativo']` limita a gravação aos campos indicados. Isso não valida o registro inteiro nem chama seu serializer. Apenas informa quais campos devem participar dessa atualização do model.

A tela atual de ingredientes não chama `axios.delete`. O botão com X chama o endpoint `toggle_ativo`. Logo, a regra acima existe no backend, mas não é o caminho percorrido por aquele botão.

**19. Ativar e desativar ingrediente é uma ação adicional, com confirmação em duas requisições.**

```python
@action(detail=True, methods=['post'])
def toggle_ativo(self, request, pk=None):
    ingrediente = self.get_object()
    confirmar = request.data.get('confirmar', False) is True
```

O decorador registra uma ação HTTP adicional. `request` contém a requisição. `pk=None` declara um parâmetro com valor padrão; na rota de detalhe o identificador normalmente é fornecido pela URL. Embora o método não use `pk` diretamente, `get_object()` usa os argumentos da rota disponíveis na view.

`.get('confirmar', False)` usa `False` quando a chave não veio. `is True` exige o booleano verdadeiro; a string `"true"` não é o mesmo valor. O frontend envia `{ confirmar }`, um booleano JavaScript convertido em booleano JSON.

```python
if (
    ingrediente.ativo
    and ingrediente.pizzas.filter(ativo=True).exists()
    and not confirmar
):
    return Response(..., status=status.HTTP_409_CONFLICT)
```

As três condições precisam ser verdadeiras: ingrediente está ativo, existe pizza ativa vinculada e a confirmação não veio. Nesse caso, o método devolve uma mensagem e `confirmacao_necessaria=True`. O `return` encerra a função antes da alteração.

Se a operação prosseguir:

```python
ingrediente.ativo = not ingrediente.ativo
ingrediente.save(update_fields=['ativo'])
serializer = self.get_serializer(ingrediente)
return Response(serializer.data, status=status.HTTP_200_OK)
```

`not` inverte o booleano. A gravação ocorre em `ingrediente.save`. O serializer é criado depois apenas para produzir a resposta. Não existe `serializer.is_valid()` nessa ação.

O serializer de ingredientes não impõe suas validações normais a esse caminho, porque elas não foram chamadas. A lógica dessa operação está no método da ação.

Na tela, uma resposta 409 chega ao `.catch(...)` do Axios. O frontend abre `window.confirm(dados.mensagem)`. Se o usuário confirmar, chama a mesma função com `confirmar=true`, fazendo uma segunda requisição. Se cancelar, retorna sem reenviar.

O bloqueio do `DELETE` e a confirmação do toggle têm critérios diferentes: o primeiro considera qualquer pizza vinculada; o segundo pede confirmação ao desativar ingrediente ligado a pizza ativa. Reativar não entra nesse pedido de confirmação.

Como a ação inverte o valor atual, dois envios consecutivos podem inverter duas vezes. Isso explica por que, se futuramente você quiser uma operação que garanta um estado específico, poderá preferir receber explicitamente o estado desejado.

**20. Adicionar estoque soma uma entrada e recalcula a média ponderada.**

Sua ação `adicionar_estoque` recupera o ingrediente, lê `quantidade_entrada` e `custo_total_entrada`, converte ambos com `Decimal(str(...))`, soma aos valores existentes, divide o novo custo pela nova quantidade, atribui os resultados e salva.

Exemplo didático:

| Valor | Antes | Entrada | Depois |
|---|---:|---:|---:|
| Quantidade | 10 KG | 5 KG | 15 KG |
| Custo total | R$ 300 | R$ 180 | R$ 480 |
| Custo médio | R$ 30/KG | R$ 36/KG | R$ 32/KG |

O cálculo é `480 / 15 = 32`. Não é a média simples entre 30 e 36, porque as quantidades são diferentes. A fórmula depende de `custo_total` representar o valor correspondente ao estoque atual.

As atribuições `ingrediente.quantidade_estoque = ...`, `ingrediente.custo_total = ...` e `ingrediente.custoMedio = ...` mudam o objeto em memória. `ingrediente.save()` efetiva sua persistência. O serializer criado depois serve para a saída.

Esta ação não utiliza um serializer de entrada nem chama `is_valid()`. Portanto, `validate_quantidade_estoque` e `validate_custo_total` não protegem automaticamente `quantidade_entrada` e `custo_total_entrada`. São nomes e caminho de execução diferentes.

Pelo código recebido, faltam tratamentos explícitos para valor ausente, texto inválido, entrada negativa, números não finitos, resultado com quantidade zero e entradas simultâneas no mesmo saldo. Uma chave ausente produz `None`; `Decimal(str(None))` tenta converter `'None'`, o que pode causar uma exceção em vez de um erro de formulário organizado.

Nenhuma das duas telas analisadas faz uma chamada a `/adicionar_estoque/`. O endpoint está definido no backend, mas não possui uma interface correspondente nesses componentes.

**21. CREATE de pizzas precisa salvar o cadastro e as linhas da composição.**

O início se parece com o cadastro de ingredientes: Axios faz `POST`; router seleciona `create`; a ação herdada instancia `PizzasSerializer`; `is_valid` valida os campos e os serializers dos itens; a ação chama `perform_create` da view.

```python
def perform_create(self, serializer):
    pizza = serializer.save()
    self.calcular_custo_producao(pizza)
```

`serializer.save()` devolve a instância criada. O retorno é atribuído à variável local `pizza`.

Como esse serializer não tinha uma instância existente, o `.save()` chama o seu `PizzasSerializer.create`:

```python
def create(self, validated_data):
    composicao_data = validated_data.pop('composicao', [])
    pizza = Pizzas.objects.create(**validated_data)
    self._salvar_composicao(pizza, composicao_data)
    return pizza
```

`validated_data` é um dicionário. `.pop('composicao', [])` retira a lista da composição e a devolve. Se a chave não existir, usa uma lista vazia como valor padrão. O método altera o dicionário de que está retirando a chave.

Essa remoção separa os campos simples da pizza dos dados das associações. Você não pode tratar a lista de composições aninhadas como se fosse uma coluna simples a ser inserida diretamente no registro da pizza.

`**validated_data` desempacota as chaves do dicionário como argumentos nomeados. Se ele contiver `{'nome': 'Mussarela', 'precoVenda': Decimal('45.00'), 'tamanho': 'G'}`, a ideia da chamada é `Pizzas.objects.create(nome='Mussarela', precoVenda=..., tamanho='G')`.

`objects.create` faz o cadastro da pizza e devolve a instância com ID. Agora as associações podem apontar para uma pizza existente.

```python
def _salvar_composicao(self, pizza, composicao_data):
    pizza.composicao.all().delete()
    custo_total = Decimal('0.00')

    for item in composicao_data:
        ingrediente = item['ingrediente']
        quantidade = item['quantidade']
        PizzaIngrediente.objects.create(
            pizza=pizza,
            ingrediente=ingrediente,
            quantidade=quantidade,
        )
        custo_total += Decimal(str(quantidade)) * ingrediente.custoMedio

    pizza.custoProducao = custo_total
    pizza.save(update_fields=['custoProducao'])
```

O `_` inicial em `_salvar_composicao` é uma convenção de método auxiliar interno; Python não impede seu acesso apenas por esse nome. Esse método não é uma ação automática do DRF. Ele roda porque você o chama explicitamente.

`pizza.composicao.all().delete()` exclui as linhas intermediárias daquela pizza. Não exclui a pizza nem os ingredientes. Em um cadastro recém-criado, normalmente não existe nenhuma linha anterior para excluir.

`for item in composicao_data` percorre a lista de dicionários validados. Em cada repetição, `item` referencia uma linha. `item['ingrediente']` já é um objeto de ingrediente, graças ao `PrimaryKeyRelatedField`. `item['quantidade']` já passou pela validação decimal.

Em `pizza=pizza`, o nome à esquerda é o argumento do campo; o valor à direita é a variável que guarda o objeto. Os nomes iguais não tornam a expressão uma comparação. A mesma explicação vale para `ingrediente=ingrediente` e `quantidade=quantidade`.

`PizzaIngrediente.objects.create(...)` grava uma associação em cada volta. `custo_total += ...` soma o custo daquela linha ao acumulador; equivale a atribuir `custo_total = custo_total + ...`.

Se 0,200 KG de mussarela custam R$ 30 por KG, essa linha custa `0,200 × 30 = 6`. Se 0,300 KG de massa custam R$ 10 por KG, custa `3`. A composição resulta em custo de produção de R$ 9.

Após o laço, o método salva o custo no registro da pizza. O `create` retorna a pizza. O `serializer.save()` devolve essa mesma instância à view. A view chama `calcular_custo_producao` novamente. Depois, a ação herdada monta a resposta 201.

O cálculo do custo está, portanto, duplicado nessa operação: ocorre no auxiliar do serializer e novamente na view. Isso é o comportamento real desse código, não duas etapas matemáticas diferentes necessárias para a fórmula.

**22. UPDATE de pizzas atualiza os campos e pode substituir toda a composição.**

O frontend de pizzas utiliza `PUT` quando `modoEdicao` está ativo e existe `pizzaSelecionada`.

Após recuperar a pizza e validar a entrada, a ação herdada chama seu `perform_update`, que faz `pizza = serializer.save()` e recalcula o custo pela view. Como há uma instância existente, o `.save()` chama este método do serializer:

```python
def update(self, instance, validated_data):
    composicao_data = validated_data.pop('composicao', None)

    for attr, value in validated_data.items():
        setattr(instance, attr, value)

    instance.save()

    if composicao_data is not None:
        self._salvar_composicao(instance, composicao_data)

    return instance
```

`instance` é a pizza recuperada do banco. `.pop('composicao', None)` separa a composição, mas agora usa `None` para distinguir “não enviada” de “uma lista enviada”. Essa distinção é útil em `PATCH`.

`validated_data.items()` permite iterar sobre pares de chave e valor. Por exemplo, `attr` recebe `'nome'` e `value` recebe `'Mussarela especial'`.

`setattr(instance, attr, value)` atribui dinamicamente um atributo. Nesse exemplo equivale a `instance.nome = 'Mussarela especial'`. Na próxima volta, pode atribuir `precoVenda`. Isso evita escrever uma atribuição separada para cada campo simples.

`instance.save()` grava os campos da pizza. Se a composição foi enviada, `_salvar_composicao` remove as linhas antigas e cria novas. A pizza mantém seu ID; os ingredientes mantêm seus IDs; as linhas intermediárias recriadas recebem novos IDs.

Portanto, remover um ingrediente no formulário e salvar a pizza não exclui o ingrediente do cadastro geral. Apenas deixa de recriar aquela associação na composição final.

Um `PATCH` só com o nome pode manter a composição porque ela não foi enviada. Já o seu frontend atual envia a lista inteira por `PUT`. Se houver um ingrediente inativo nessa lista, o `PrimaryKeyRelatedField` atual pode rejeitar a edição antes de `update` ser executado.

Não há um `transaction.atomic()` explícito envolvendo a substituição da composição nesses arquivos. Sem uma transação externa configurada, uma falha depois de excluir linhas e antes de recriar todas pode deixar uma alteração parcial. Um serializer ter validado a entrada não elimina falhas de persistência. Essa é uma razão concreta para proteger essa operação com uma transação ao evoluir o código.

**23. A view de pizzas recalcula custos também durante consultas.**

```python
def get_queryset(self):
    queryset = Pizzas.objects.all().order_by('id')
    for pizza in queryset:
        self.calcular_custo_producao(pizza)
    return queryset
```

A consulta seleciona todas as pizzas, inclusive inativas. O `for` faz a coleção ser percorrida e chama o cálculo para cada pizza antes de devolver o `QuerySet`.

```python
itens = PizzaIngrediente.objects.filter(
    pizza=pizza_selecionada
).select_related('ingrediente')
```

O filtro busca somente linhas daquela pizza. `select_related('ingrediente')` permite trazer os ingredientes relacionados junto da consulta dessas linhas, evitando uma busca adicional por ingrediente em cada acesso dessa rotina.

O método percorre os itens, multiplica `item.quantidade` por `item.ingrediente.custoMedio`, soma, atribui a `pizza_selecionada.custoProducao`, salva esse campo e devolve o total.

Esse retorno poderia ser utilizado por quem chama, mas no seu `get_queryset` e nos ganchos de gravação ele não é armazenado em outra variável. O efeito aproveitado é a atualização do custo da pizza.

Consequência: `GET /api/pizzas/` pode escrever no banco. Além disso, operações individuais usam `get_object()`, que parte de `get_queryset()`. Até consultar, editar, desativar ou excluir uma pizza específica pode recalcular todas as pizzas antes da busca do alvo.

Isso mantém o custo atualizado com os custos médios no momento dessa consulta, mas mistura leitura com gravação e aumenta o número de operações no banco. Também não significa que a tela já aberta será atualizada imediatamente quando um ingrediente mudar: ela precisa buscar novamente os dados ou atualizar seu estado por outro mecanismo.

**24. DELETE e toggle de pizzas têm resultados diferentes.**

`PizzasViewSet` não sobrescreve `perform_destroy`. Assim, `DELETE /api/pizzas/7/` usa a exclusão física padrão: o registro da pizza é apagado pelo ORM, e as suas linhas de composição são alcançadas por `CASCADE`, considerando os models enviados.

Já a ação escrita por você faz:

```python
pizza = self.get_object()
pizza.ativo = not pizza.ativo
pizza.save(update_fields=['ativo'])
serializer = self.get_serializer(pizza)
return Response(serializer.data)
```

Essa ação apenas inverte o status. Como não foi passado outro status a `Response`, a resposta normal é 200. Não existe confirmação semelhante à ação de ingredientes.

A tela atual chama essa ação para o botão de status. Ela não chama `DELETE`.

| Operação | Ingredientes | Pizzas |
|---|---|---|
| `DELETE /{id}/` | Bloqueia se houver pizza vinculada; caso contrário, desativa | Exclui fisicamente pelo comportamento herdado |
| `POST /{id}/toggle_ativo/` | Inverte status; pode exigir confirmação | Inverte status |
| Botão de status na tela atual | Chama toggle | Chama toggle |

**25. Este é o mapa exato das suas sobrescritas.**

| Método | Ingredientes | Pizzas |
|---|---|---|
| `ViewSet.list` | Herdado | Herdado |
| `ViewSet.retrieve` | Herdado | Herdado |
| `ViewSet.create` | Herdado | Herdado |
| `ViewSet.update` | Herdado | Herdado |
| `ViewSet.partial_update` | Herdado; gancho atual não lida bem com omissões | Herdado; serializer distingue composição omitida |
| `ViewSet.destroy` | Herdado | Herdado |
| `ViewSet.get_queryset` | Sobrescrito: todos, ordenados por ID | Sobrescrito: todos, com recálculo e gravação |
| `ViewSet.perform_create` | Sobrescrito: calcula custo médio | Sobrescrito: salva e recalcula custo |
| `ViewSet.perform_update` | Sobrescrito: recalcula custo médio | Sobrescrito: salva e recalcula custo |
| `ViewSet.perform_destroy` | Sobrescrito: bloqueio e desativação | Herdado: exclusão física |
| `Serializer.create` | Herdado | Sobrescrito: cadastro e composição |
| `Serializer.update` | Herdado | Sobrescrito: campos e possível substituição da composição |
| `Model.save` | Herdado | Herdado |

`perform_create` e `perform_update` são pontos de personalização chamados pelas ações de criação e atualização. Já `perform_destroy` recebe a instância a excluir. Essas separações permitem acrescentar comportamento preservando a organização HTTP herdada. [Ganchos das views genéricas](https://www.django-rest-framework.org/api-guide/generic-views/).

Nenhum `save()` do model foi sobrescrito nos arquivos recebidos. Logo, suas regras de cálculo não passam a existir automaticamente em todo lugar que chamar `Ingredientes(...).save()` ou `Pizzas(...).save()`.

Uma gravação direta no model também não chama automaticamente `serializer.is_valid()`. E `Model.save()` não chama automaticamente `full_clean()`. Restrições do banco continuam existindo, mas suas validações personalizadas da API só rodam quando o fluxo as invoca. [Validação e persistência de instâncias no Django](https://docs.djangoproject.com/en/5.2/ref/models/instances/).

**26. No frontend, seus componentes são funções, e os tipos descrevem os dados.**

```tsx
import React, { useEffect, useState } from 'react'
import axios from 'axios'
import { InputText } from 'primereact/inputtext'
import { Button } from 'primereact/button'
import { DataTable } from 'primereact/datatable'
import { Column } from 'primereact/column'
import { Dialog } from 'primereact/dialog'
import Navbar from '../components/Navbar'
```

`React` é o import padrão do pacote; os nomes entre chaves são imports nomeados. `useState` e `useEffect` são funções do React chamadas hooks. Seus arquivos também usam `React.ChangeEvent` e `React.FormEvent` como tipos de eventos.

Axios é usado para HTTP. `InputText`, `Button`, `DataTable`, `Column` e `Dialog` são componentes de interface fornecidos pelo PrimeReact. `Navbar` é um componente do próprio projeto, importado de um caminho relativo: `..` sobe um diretório, depois entra em `components`.

```tsx
function GerenciarIngredientes() {
  // Estados, funções e retorno da interface.
}

export default GerenciarIngredientes
```

A declaração cria uma função de componente. O React a executa ao renderizar a tela e volta a executá-la quando precisa refletir atualizações de estado. `export default` disponibiliza essa função como a exportação principal do módulo, permitindo que outro arquivo a importe e utilize como `<GerenciarIngredientes />`.

Essa função não é uma classe Python nem uma tabela. Os `type Ingrediente`, `type Pizza` e similares também não são classes instanciadas com `new`: são descrições de tipos para o TypeScript. Eles ajudam a verificar o código durante desenvolvimento/compilação e não criam validação automática dos dados recebidos pela rede.

`.tsx` é uma extensão usada para TypeScript que contém JSX, a sintaxe de interface com elementos como `<Button />`. JSX é transformado pela ferramenta de construção do frontend em código que o React pode utilizar.

O corpo da função do componente é executado na renderização; o corpo de um callback de clique só é executado quando alguém o chama. Declarar `const salvarEdicao = (...) => { ... }` durante uma renderização não envia um `PUT` por si só.

**27. Entenda cada tipo de ingrediente e o problema da definição de formulário.**

```tsx
type Ingrediente = {
  id: number
  nome: string
  quantidade_estoque: number | string
  unidade_De_Medida: string
  custo_total: number | string
  custoMedio: number | string
  estoque_minimo: number | string
  ativo: boolean
}
```

`number` é número JavaScript; `string` é texto; `boolean` é verdadeiro ou falso. O `|` define uma união: `number | string` permite qualquer um desses dois tipos. Isso acomoda, por exemplo, valores decimais recebidos como strings pela API.

O TypeScript não transforma `'30.00'` em `30` porque você declarou uma união. A conversão ocorre nas chamadas explícitas a `Number(...)` ou em outra função escrita para isso.

```tsx
type FormIngrediente = Omit<Ingrediente, 'id'> & {
  quantidade_estoque: string
  custo_total: string
  estoque_minimo: string
}
```

`Omit<Ingrediente, 'id'>` constrói um tipo derivado sem o campo `id`. O `&` cria uma interseção entre as exigências dos dois tipos. Para os três campos repetidos, a combinação de `number | string` com `string` resulta na exigência de string. Isso é útil porque um input pode estar vazio ou conter texto durante a digitação.

Porém, seu `Omit` retira **somente** `id`. Os campos `custoMedio` e `ativo` continuam obrigatórios. O objeto inicial passado a `useState<FormIngrediente>` e o objeto de `limparFormulario` não fornecem esses dois campos. Assim, a definição apresentada tem uma incompatibilidade de tipos verificável no próprio código.

Uma definição coerente com um formulário que não edita campos calculados/de status deveria também excluí-los ou declarar explicitamente apenas os campos editáveis. Exemplo de direção de ajuste, não aplicado ao arquivo:

```tsx
type FormIngrediente = Omit<
  Ingrediente,
  'id' | 'custoMedio' | 'ativo'
> & {
  quantidade_estoque: string
  custo_total: string
  estoque_minimo: string
}
```

Não é necessário obrigar o usuário a preencher campos somente de leitura para satisfazer a tipagem do formulário.

```tsx
type EditorOptions = {
  value: string | number
  editorCallback?: (value: string) => void
}
```

`EditorOptions` descreve as opções utilizadas pelos seus editores inline. `value` é o valor atual da célula. `editorCallback?` é uma propriedade opcional. Se existir, é uma função que recebe uma string e cujo retorno não é utilizado (`void`).

```tsx
type ErrosIngrediente = Partial<
  Record<
    'nome' | 'unidade_De_Medida' | 'quantidade_estoque'
    | 'custo_total' | 'estoque_minimo',
    string
  >
>
```

`Record` descreve um objeto com essas chaves e mensagens de texto. `Partial` torna as propriedades opcionais, porque um formulário pode ter erro somente no nome ou não ter erro algum. Por isso `{}` é um valor válido para esse estado. Essas operações são utilidades de tipos, não chamadas ao backend. [Tipos utilitários do TypeScript](https://www.typescriptlang.org/docs/handbook/utility-types.html).

**28. `useState` fornece um valor atual e uma função para solicitar sua atualização.**

```tsx
const [ingredientes, setIngredientes] = useState<Ingrediente[]>([])
```

`const` declara uma referência que não será reatribuída naquele escopo. Os colchetes à esquerda fazem desestruturação: retiram os dois valores retornados pelo hook. `ingredientes` é o estado atual; `setIngredientes` é a função que solicita outro estado.

`<Ingrediente[]>` é um argumento de tipo: o estado será um array de ingredientes. `([])` passa um array vazio como valor inicial. Os sinais `<>` e `()` têm funções diferentes.

Quando o setter é chamado, o React organiza uma atualização e renderiza usando o novo estado. A variável da renderização em execução não é transformada imediatamente no novo valor; uma próxima renderização receberá o estado atualizado. [Referência de `useState`](https://react.dev/reference/react/useState).

| Estado de ingredientes | Valor inicial | Uso |
|---|---|---|
| `ingredientes` | `[]` | Dados exibidos na tabela |
| `ingredienteSelecionado` | `null` | Objeto selecionado na tabela |
| `popupCadastroAberto` | `false` | Visibilidade do modal de cadastro |
| `errosCadastro` | `{}` | Mensagens de validação |
| `novoIngrediente` | Objeto de campos vazios | Valores digitados no cadastro |

`Ingrediente | null` permite ter um ingrediente selecionado ou nenhum. `useState(false)` normalmente infere booleano; `useState('')` infere string. No array vazio, o argumento genérico explicita que tipo de item poderá ser armazenado.

Estado não é banco de dados. Atualizar `novoIngrediente` não grava nada no servidor. Fechar ou recarregar a página normalmente perde o que estava apenas nesse estado, enquanto os registros já persistidos continuam no banco.

**29. `prev`, spread e propriedades computadas atualizam o formulário.**

```tsx
setNovoIngrediente(prev => ({
  ...prev,
  [name]: valor,
}))
```

Você está passando uma função ao setter. O React chama essa função com o estado anterior relevante para a atualização. O nome `prev` é escolhido por você; não é uma palavra reservada.

`...prev` copia as propriedades do objeto anterior para um objeto novo. `[name]` é uma chave computada: usa o conteúdo da variável `name` como nome da propriedade. Se `name` for `'nome'`, a expressão atualiza `nome`. Se for `'estoque_minimo'`, atualiza esse campo.

A ordem importa. Como `[name]: valor` vem depois de `...prev`, o novo valor substitui o valor anterior daquele campo, preservando os demais.

Os parênteses em `prev => ({ ... })` permitem devolver diretamente um objeto em uma arrow function. Sem eles, as chaves poderiam ser interpretadas como corpo de função, que precisaria de `return`.

Esse padrão cria um objeto novo em vez de modificar o estado anterior diretamente. A cópia é superficial: objetos e arrays internos precisam de atualização própria quando são alterados. No formulário de pizzas, é por isso que você também utiliza `map`, `filter` e um novo array para a composição.

`const` não torna automaticamente todas as propriedades internas de um objeto imutáveis. A escolha de criar novas estruturas é uma disciplina de atualização de estado, não uma garantia fornecida apenas pela palavra `const`.

**30. Os handlers de digitação recebem eventos e alteram estado local.**

```tsx
const handleInputChange = (
  e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>
) => {
  const { name, value } = e.target
  const valor = ['quantidade_estoque', 'estoque_minimo'].includes(name)
    && Number(value) < 0
      ? ''
      : value

  setNovoIngrediente(prev => ({ ...prev, [name]: valor }))
}
```

`e` é o evento recebido do input ou select. A anotação TypeScript informa os tipos de elemento que esse handler aceita. `e.target` é o alvo do evento. A desestruturação extrai `name` e `value`.

O atributo `name` do input faz a conexão com a chave do objeto. Por exemplo, `name="quantidade_estoque"` faz `[name]` atualizar exatamente essa propriedade.

`.includes(name)` verifica se o campo é um dos numéricos protegidos. `Number(value) < 0` detecta o valor negativo convertido. O operador ternário `condição ? A : B` escolhe `''` para limpar a entrada negativa, ou mantém o texto recebido.

Mesmo um input `type="number"` fornece normalmente seu `value` como string. Isso permite representar o campo vazio, que não é a mesma coisa que zero. As conversões para número são feitas explicitamente antes da requisição.

Na tela de pizzas, `handleInputChange` é mais simples: recebe o evento, extrai `name` e `value`, e atualiza `novaPizza`. Ele é utilizado no nome e tamanho. A quantidade de cada ingrediente é tratada por `alterarIngrediente`, porque pertence a um array interno.

**31. Máscara de moeda e conversão para número fazem trajetos opostos.**

`handleMoedaChange` existe nos dois componentes com o mesmo princípio:

```tsx
let valorNum = value.replace(/\D/g, '')
const options = {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
}
const valorFinal = (Number(valorNum) / 100)
  .toLocaleString('pt-BR', options)
```

`replace` substitui os trechos que correspondem à expressão regular. `\D` representa caracteres que não são dígitos; `g` manda procurar todas as ocorrências. Assim, letras, espaços e pontuação são removidos.

O código interpreta os dígitos como centavos. Digitando `1234`, divide por 100 e obtém `12.34`. `toLocaleString('pt-BR', options)` transforma em texto como `12,34`. Depois o estado recebe a template string `` `R$ ${valorFinal}` ``.

Se não restarem dígitos, o campo é deixado vazio e o `return` encerra o handler. `let` permite reatribuição, mas no trecho enviado `valorNum` não é reatribuído; poderia ser `const` sem mudar essa lógica.

`converterMoedaParaNumero` faz o caminho para a requisição:

1. Se recebeu um `number`, devolve o próprio número.
2. Se recebeu texto, remove espaços externos com `trim()`.
3. Se ficou vazio, devolve `NaN`.
4. Se encontrar `R$` ou vírgula, mantém apenas dígitos e divide por 100.
5. Caso contrário, utiliza `Number(valorLimpo)`.

| Entrada | Resultado da função atual |
|---|---:|
| `45` | `45` |
| `'45.00'` | `45` |
| `'R$ 45,00'` | `45` |
| `'1.234,56'` | `1234.56` |
| `''` | `NaN` |

Essa função pressupõe o formato produzido pela sua máscara. Não é um interpretador geral de qualquer número brasileiro: por exemplo, `'12,3'` seria tratado como 123 centavos, resultando em 1,23. Ao remover tudo que não é dígito, também remove um sinal negativo em texto monetário. A validação não deve depender de aceitar qualquer texto colado como se fosse necessariamente bem formado.

`NaN` significa resultado numérico inválido. `Number.isFinite(...)` permite rejeitar `NaN` e infinitos. Isso aparece nas validações do frontend.

Já `moeda(...)`, usada nas tabelas, apenas formata um valor para exibição. No componente de ingredientes ela retorna `'-'` quando recebe nulo/indefinido, ou `R$ ` seguido de `Number(valor).toFixed(2)`.

`toFixed(2)` produz duas casas usando ponto, como `'30.00'`. Portanto, a máscara de digitação brasileira e o texto da tabela ainda não estão padronizados: a tabela normalmente mostra `R$ 30.00`, enquanto o input mascara como `R$ 30,00`.

Na tela de pizzas, a função `moeda` usa `valor ?? 0`, que substitui somente `null` ou `undefined` por zero. Depois verifica se o número é finito. Se não for, retorna `'R$ 0,00'`; se for, também utiliza `toFixed(2)`.

**32. O carregamento inicial consulta o backend pelo `useEffect`.**

Na tela de ingredientes:

```tsx
useEffect(() => {
  carregarIngredientes()
}, [])
```

O primeiro argumento é uma função a executar como efeito. O segundo é a lista de dependências; vazia, ela indica que o efeito não depende de valores reativos que devam provocar nova execução por mudança. Ele é utilizado aqui para carregar os dados quando o componente é montado.

Não é correto dizer que esse efeito executará uma única vez em qualquer circunstância: remontagens podem executá-lo novamente, e o Strict Mode pode realizar um ciclo adicional de efeito em desenvolvimento. [Referência de `useEffect`](https://react.dev/reference/react/useEffect).

`carregarIngredientes` está declarado mais abaixo como uma constante que referencia uma função. Isso funciona aqui porque a função passada ao efeito é executada depois da renderização, quando a declaração já foi processada. Chamar uma constante antes de sua inicialização durante a execução síncrona seria uma situação diferente.

```tsx
axios.get('http://localhost:8000/api/ingredientes/')
  .then(response => {
    setIngredientes(response.data)
  })
  .catch(err => {
    console.error('Erro ao carregar ingredientes:', err)
  })
```

`axios.get` inicia a requisição e devolve uma Promise, que representa sua conclusão futura. `.then` registra o que fazer em caso de sucesso. `.catch` registra o que fazer em caso de falha. O navegador não precisa parar toda a interface esperando a resposta.

`response.data` contém o corpo interpretado da resposta. `setIngredientes` coloca a lista no estado, e o React renderiza a tabela com ela. O `.catch` atual só escreve o erro no console da tela de ingredientes; não mostra uma mensagem específica de carregamento ao usuário.

Na tela de pizzas, o efeito inicia dois `GET`: um de pizzas e outro de ingredientes. Como não existe `await` nem um `.then` envolvendo o início da segunda requisição, elas podem estar em andamento ao mesmo tempo. Cada uma atualiza seu próprio estado quando terminar.

`axios.get<Pizza[]>(...)` informa ao TypeScript o tipo esperado para `response.data`. O Axios não verifica, por esse argumento genérico, se o servidor realmente retornou uma lista válida de pizzas. O tipo é uma expectativa estática do seu código.

**33. A validação de ingrediente no frontend produz mensagens de interface.**

```tsx
const validarIngrediente = (
  ingrediente: FormIngrediente | Ingrediente
) => {
  const erros: ErrosIngrediente = {}
  // Verificações...
  setErrosCadastro(erros)
  return Object.keys(erros).length === 0
}
```

O parâmetro aceita tanto um formulário quanto um ingrediente existente, permitindo reutilizar a função no cadastro e na edição inline. `erros` começa vazio; cada reprovação acrescenta uma mensagem em uma chave.

`Object.keys(erros)` devolve uma lista das propriedades existentes. Se o tamanho for zero, não foi registrado erro e a função retorna `true`.

Suas regras são: nome não vazio após `trim`; unidade preenchida; quantidade preenchida, finita e não negativa; custo informado e não negativo quando a quantidade é positiva; estoque mínimo preenchido, finito e não negativo.

Você testa string vazia separadamente porque `Number('')` resulta em zero. Sem esse cuidado, um campo vazio poderia passar como se o usuário tivesse informado zero.

Quando quantidade é zero, essa função não exige custo. Ela também não rejeita, nesse ramo, todo custo inválido que tenha sido fornecido. A validação do backend continua necessária.

`setErrosCadastro` apenas solicita atualização visual. O retorno booleano é o que permite ao chamador interromper a gravação imediatamente. A função não manda uma requisição e não chama o serializer Python.

Ao cadastrar, `if (!validarIngrediente(novoIngrediente)) return` chama a validação, inverte o resultado com `!` e encerra a função se houver erro.

**34. Cadastrar ingrediente monta um payload e usa a resposta real do servidor.**

Depois de validar, `cadastrarIngrediente` cria `ingredienteParaEnviar`. O nome passa por `trim`; quantidades e mínimo passam por `Number`; o custo passa pelo conversor de moeda.

Esse objeto é o **payload**, isto é, o conteúdo enviado no corpo da requisição. Ele contém apenas os campos editáveis usados no cadastro. O frontend não precisa inventar um ID ou calcular `custoMedio`.

`axios.post(url, ingredienteParaEnviar)` passa a URL como primeiro argumento e o corpo como segundo. A persistência efetiva depende de o backend concluir seu fluxo. O fato de a função JavaScript ter sido chamada ainda não significa que o banco já salvou.

```tsx
setIngredientes(prev => [
  ...prev,
  response.data,
])
```

No sucesso, cria-se um novo array com os registros anteriores e o objeto devolvido pelo backend. Esse objeto já contém o ID, o custo médio calculado e o status. A tela não faz outro `GET` nessa operação: atualiza localmente a lista usando a resposta.

`limparFormulario()` redefine os valores para strings vazias, remove erros e fecha o diálogo. A mesma função é reutilizada no cancelamento e no fechamento do modal.

Quando custo está vazio e quantidade é zero, o conversor retorna `NaN`. No caminho usual de serialização JSON do Axios, um `NaN` em propriedade de objeto vira `null`. O backend atual aceita esse nulo e o transforma em zero nesse cadastro. Seria mais explícito preparar deliberadamente `null`, zero ou omissão segundo o contrato escolhido, em vez de depender dessa conversão indireta.

No `.catch`, `err.response?.data || err` tenta mostrar os dados retornados pelo servidor e, se não houver, mostra o erro original. `?.` evita acessar uma propriedade de algo nulo/indefinido. A ausência de `response` pode acontecer em falhas de rede.

O seu tratamento visual verifica somente `err.response?.data?.nome`. Se existir, usa a primeira mensagem da lista com `[0]`. Se não, coloca uma mensagem genérica na chave `nome`. Assim, um erro de custo ou quantidade devolvido pelo backend não é necessariamente mostrado junto ao campo correspondente, mesmo que o servidor tenha identificado esse campo corretamente.

**35. A edição inline usa um rascunho controlado pela DataTable antes do `PUT`.**

Estas propriedades habilitam o fluxo:

```tsx
<DataTable
  value={ingredientes}
  dataKey="id"
  editMode="row"
  onRowEditComplete={salvarEdicao}
>
  <Column field="nome" editor={options => textEditor(options)} />
  <Column rowEditor header="Editar" />
</DataTable>
```

`value` fornece os registros. `dataKey` identifica cada registro pelo ID. `editMode="row"` configura edição por linha. `rowEditor` acrescenta os controles de edição da linha. A propriedade `editor` de cada coluna diz qual componente utilizar quando aquele campo estiver sendo editado.

O PrimeReact chama seu editor com `options`. Você não constrói manualmente esse objeto na tela; a biblioteca o fornece.

```tsx
const textEditor = (options: EditorOptions) => (
  <InputText
    value={String(options.value)}
    onChange={e => options.editorCallback?.(e.target.value)}
  />
)
```

`String(...)` converte o valor atual para texto. `options.editorCallback?.(...)` chama a função de atualização do rascunho da edição, se ela existir. O `?.` antes dos parênteses é uma chamada opcional.

Esse callback não salva no banco. Ele comunica à DataTable o valor digitado. É um nível diferente de `setNovoIngrediente`, que controla o formulário de cadastro.

`numberEditor` segue o mesmo princípio, mas usa `type="number"`, `min="0"` e transforma entradas negativas em texto vazio antes de chamar o callback.

Quando a edição da linha é concluída, a tabela chama `salvarEdicao(e)`. `e.newData` contém o objeto com os valores editados. O parâmetro foi declarado `any`, o que abre mão da checagem detalhada desse evento pelo TypeScript.

O fluxo concreto de `salvarEdicao` é:

1. Obtém `ingredienteEditado = e.newData`.
2. Chama `validarIngrediente(ingredienteEditado)`.
3. Monta os campos que serão enviados.
4. Faz `PUT` para a URL com `ingredienteEditado.id`.
5. Espera a resposta pelo `.then`.
6. Substitui a linha daquele ID no array.
7. Limpa a seleção com `setIngredienteSelecionado(null)`.

```tsx
setIngredientes(prev =>
  prev.map(ingrediente =>
    ingrediente.id === ingredienteEditado.id
      ? response.data
      : ingrediente
  )
)
```

`map` constrói outro array. Para cada ingrediente, compara seu ID com o editado. Se coincidir, devolve o objeto atualizado pelo servidor. Caso contrário, preserva o objeto daquela posição.

O nome enviado nessa função não passa por `trim`, mas a validação do backend faz a normalização. O custo da edição passa por `Number`, pois o editor numérico não utiliza a mesma máscara monetária do cadastro.

No erro de backend, essa função utiliza `alert` para nome duplicado ou mensagem genérica. Já falhas na validação local preenchem `errosCadastro`, que é exibido no modal de cadastro. Durante edição inline, esse modal pode estar fechado; por isso o erro local pode não ficar visível junto à linha. Também não há um `rowEditValidator` configurado nesse trecho para manter a edição a partir de uma regra da própria tabela.

A DataTable organiza os eventos e os editores; o `PUT` continua sendo responsabilidade de sua função. [Tabela e edição de linhas do PrimeReact](https://v10.primereact.org/datatable/).

**36. A seleção de uma linha é diferente de editar uma linha.**

```tsx
selection={ingredienteSelecionado}
onSelectionChange={e =>
  setIngredienteSelecionado(e.value as Ingrediente | null)
}
selectionMode="single"
```

`selection` fornece à tabela a seleção atual. `selectionMode="single"` permite um registro selecionado por vez. Quando o usuário muda a seleção, a biblioteca chama `onSelectionChange` e coloca o valor no evento `e.value`.

`as Ingrediente | null` é uma afirmação de tipo para o TypeScript. Não converte dados em tempo de execução e não valida se o conteúdo possui todos os campos de `Ingrediente`.

A coluna com `selectionMode="single"` mostra o controle de seleção. Isso não é o botão de edição. A edição é habilitada por `editMode="row"`, editores de campo e `rowEditor`.

Da mesma maneira, os botões de status recebem o objeto da própria linha por meio de `body={acoesDaLinha}`. Eles não dependem de `ingredienteSelecionado` para saber qual ID alterar.

**37. O toggle de ingrediente tem parâmetros, requisição, repetição confirmada e atualização visual.**

```tsx
const alternarStatusIngrediente = (
  ingrediente: Ingrediente,
  confirmar: boolean = false
) => {
  axios.post(
    `http://localhost:8000/api/ingredientes/${ingrediente.id}/toggle_ativo/`,
    { confirmar }
  )
  // Tratamento da Promise...
}
```

O primeiro argumento é o ingrediente da linha. O segundo é opcional na chamada porque tem valor padrão `false`. `{ confirmar }` é abreviação de `{ confirmar: confirmar }`.

A template string monta a URL inserindo `ingrediente.id` entre `${...}`. Se o ID for 7, o endpoint será `/api/ingredientes/7/toggle_ativo/`.

No sucesso, a função usa `map` para substituir a linha pela resposta. No erro, verifica simultaneamente o status 409, a indicação `confirmacao_necessaria === true` e se ainda não estava confirmando.

`window.confirm` abre o diálogo nativo do navegador e devolve booleano. Se o usuário aceitar, `alternarStatusIngrediente(ingrediente, true)` inicia a nova chamada com confirmação. É uma chamada da função a si mesma, disparada depois da resposta, com o segundo argumento alterado.

O `return` desse ramo impede que a mensagem genérica de erro apareça logo depois do diálogo de confirmação. Se o usuário cancelar, também encerra o tratamento sem alterar o status.

`acoesDaLinha` é uma função que recebe o ingrediente e devolve JSX com um botão. `body={acoesDaLinha}` passa a referência da função à coluna; a tabela a chama para cada linha.

| Propriedade do botão | Efeito |
|---|---|
| `icon` | X para desativar o ativo; check para ativar o inativo |
| `rounded` | Formato arredondado |
| `text` | Aparência de botão de texto |
| `severity` | Variante visual de aviso ou sucesso |
| `aria-label` | Nome acessível com a ação e o ingrediente |
| `tooltip` | Texto de ajuda mostrado na interação correspondente |
| `onClick` | Função executada ao clicar |

`onClick={() => alternarStatusIngrediente(ingrediente)}` passa uma função, que guarda acesso ao ingrediente daquela renderização. Ela será executada no clique. Escrever `onClick={alternarStatusIngrediente(ingrediente)}` executaria a função durante a renderização e passaria seu retorno ao lugar errado. [Eventos no React](https://react.dev/learn/responding-to-events).

**38. O retorno visual de ingredientes conecta estado, formulário e tabela.**

O `return (...)` de `GerenciarIngredientes` descreve a interface. `<div>`, `<form>`, `<label>` e `<select>` são elementos HTML. `<Navbar>`, `<Dialog>`, `<InputText>`, `<Button>`, `<DataTable>` e `<Column>` são componentes React.

`className="management-page"` associa classes CSS. Não declara uma classe de programação. Como os arquivos de estilo não foram enviados, os nomes permitem identificar os pontos de aplicação, mas não confirmar todo o resultado visual.

`style={{ padding: '20px' }}` teria uma chave externa para inserir uma expressão JavaScript no JSX e uma chave interna para o objeto de estilos. Nos seus trechos, `display`, `justifyContent`, `gap`, `width`, `color` e `fontWeight` seguem esse padrão. Nomes CSS com hífen costumam aparecer em camelCase no objeto, como `justifyContent`.

O botão “Adicionar ingrediente” chama `setPopupCadastroAberto(true)`. Como o diálogo recebe `visible={popupCadastroAberto}`, a próxima renderização o abre.

`header="Novo ingrediente"` define o título do diálogo. `onHide={limparFormulario}` informa o que fazer quando houver pedido de fechamento. `modal` é uma propriedade booleana equivalente a `modal={true}`. `width: 'min(500px, 92vw)'` utiliza a função CSS `min`: escolhe a menor largura entre 500 pixels e 92% da largura da janela.

O formulário usa:

```tsx
onSubmit={event => {
  event.preventDefault()
  cadastrarIngrediente()
}}
```

`preventDefault()` impede o envio HTML tradicional que navegaria/recarregaria a página. Depois, sua função controla o envio pelo Axios. O botão `type="submit"` dispara esse fluxo; o botão `type="button"` de cancelar executa apenas seu clique.

Em cada input, `value={novoIngrediente.nome}` lê do estado e `onChange={handleInputChange}` atualiza esse estado. Esse é um campo controlado. `id` identifica o elemento; `htmlFor` do label liga o rótulo ao input; `name` indica a chave tratada pelo handler; `placeholder` mostra uma dica quando vazio.

```tsx
className={errosCadastro.nome ? 'p-invalid' : ''}
{errosCadastro.nome && (
  <small className="field-error">{errosCadastro.nome}</small>
)}
```

O ternário escolhe uma classe de erro. O `&&` faz uma renderização condicional: havendo mensagem, inclui o `<small>` com o texto. Isso não é um novo campo do banco; é estado da apresentação.

`min="0"` é uma restrição do elemento numérico, mas não substitui a validação JavaScript nem a do servidor. Nos inputs de quantidade e mínimo do cadastro de ingredientes não foi definido `step`; o comportamento nativo padrão pode não aceitar frações no envio do formulário. Para quantidades com três casas, a configuração precisa ser coerente com essa precisão.

Na tabela, `field` escolhe a propriedade do registro e `header` escolhe o título da coluna. `body` personaliza a exibição; `editor` personaliza a edição. A coluna `custoMedio` tem formatação, mas não tem editor. O status usa texto e cor condicionais.

`estoque_minimo` é cadastrado e reenviado na edição, mas não possui uma coluna de visualização/edição própria nessa tabela. A unidade, por outro lado, usa um editor textual livre na linha, embora o cadastro ofereça somente `KG`; um valor diferente será rejeitado pelo backend.

`responsiveLayout="scroll"` configura o comportamento responsivo previsto por essa propriedade da biblioteca. `headerStyle` e `bodyStyle` aplicam estilos ao cabeçalho e ao conteúdo da coluna, respectivamente.

**39. Os tipos de pizzas distinguem dados recebidos e rascunho de formulário.**

`Pizza` descreve o objeto devolvido pela API: ID, nome, preço, tamanho, custo de produção, lista de composição e status. O campo `ingredientes: PizzaIngrediente[]` declara um array de itens de composição.

`PizzaIngrediente` no TypeScript descreve o JSON de um item. Ele contém ID do ingrediente, nome, unidade, status, quantidade e custo médio. Esse tipo não é a classe Python; é uma descrição do contrato de dados recebido no navegador.

`Ingrediente` descreve a lista de ingredientes disponíveis, carregada separadamente. Há um tipo de mesmo nome no outro arquivo, mas cada módulo declara seu próprio tipo local. Eles não são automaticamente uma única definição compartilhada.

```tsx
type FormPizzaIngrediente = {
  ingrediente_id: number | ''
  quantidade: string
}
```

O ID pode ser um número ou a string literal vazia. Esse vazio representa a opção “Selecione o ingrediente”. A quantidade fica em string para acompanhar o input, inclusive quando ainda não foi preenchido.

`FormPizza` possui `nome`, `preco_de_venda` e `tamanho` em string, além do array de itens do formulário. Não inclui custo calculado nem status, porque não são preenchidos nesse modal.

Em `ErrosPizza`, cada propriedade termina com `?`, tornando a mensagem opcional. Isso equivale à intenção de um objeto de erros inicialmente vazio.

```tsx
const formularioInicial: FormPizza = {
  nome: '',
  preco_de_venda: '',
  tamanho: '',
  ingredientes: [],
}
```

Essa constante, fora do componente, serve como referência para iniciar e limpar o formulário. Os métodos escritos atualizam os dados criando novas estruturas, preservando esse objeto inicial.

**40. Cada estado de pizzas controla uma parte concreta da tela.**

| Estado | Função |
|---|---|
| `pizzas` | Lista principal exibida na tabela |
| `pizzaSelecionada` | Pizza cujo cadastro está sendo editado |
| `novaPizza` | Rascunho do formulário, usado tanto em criação quanto edição |
| `popupAberto` | Visibilidade do diálogo |
| `erro` | Mensagem geral de falha da página |
| `errosFormulario` | Erros associados aos campos |
| `ingredientesDisponiveis` | Lista recebida do endpoint de ingredientes |
| `pizzasExpandidas` | Controle das linhas expandidas da tabela |
| `modoEdicao` | Escolha entre criar com POST e editar com PUT |

`novaPizza` é um nome de variável, mas não significa que ela só possa conter uma pizza nova. Quando você abre uma edição, o código preenche esse mesmo estado com os valores existentes.

Na montagem, os dois `GET` preenchem `pizzas` e `ingredientesDisponiveis`. Se um falhar, o respectivo `.catch` registra uma mensagem em `erro`. A tela renderiza esse texto no parágrafo condicionado por `{erro && ...}`.

O carregamento inicial inclui todos os ingredientes retornados pelo backend. O filtro de ativos é aplicado posteriormente ao construir as opções do select, não nesse `GET`.

**41. Adicionar, alterar e remover uma linha de composição só modifica o rascunho.**

```tsx
const adicionarIngrediente = () => {
  setNovaPizza(prev => ({
    ...prev,
    ingredientes: [
      ...prev.ingredientes,
      { ingrediente_id: '', quantidade: '' },
    ],
  }))
}
```

O spread externo preserva nome, preço e tamanho. O spread interno preserva as linhas existentes. O novo objeto vazio é acrescentado ao final. Nenhum ingrediente é cadastrado no banco por essa função: ela adiciona um espaço para escolher um ingrediente já cadastrado.

`alterarIngrediente(index, campo, valor)` recebe três argumentos: posição da linha, nome do campo e texto recebido do input.

`campo: 'ingrediente_id' | 'quantidade'` é uma união de dois valores literais permitidos. Isso ajuda a evitar que o programador passe acidentalmente outro nome de campo.

O método faz `map` sobre a composição. Quando `indiceAtual !== index`, devolve o mesmo item, sem alterá-lo. Ao encontrar a posição certa, devolve um novo objeto com o campo modificado.

Se o campo for `ingrediente_id`, converte o valor com `Number`, exceto quando ele é `''`, preservando a opção vazia. Se o campo for quantidade, preserva a string do input.

Exemplo de chamada ao selecionar o ingrediente 7 na segunda linha:

```tsx
alterarIngrediente(1, 'ingrediente_id', '7')
```

Arrays começam na posição zero; por isso a segunda linha tem índice 1. O ID 7 e o índice 1 têm significados diferentes: um identifica o ingrediente no banco; o outro identifica a posição do rascunho no formulário.

`removerIngrediente(index)` utiliza `filter((_, indiceAtual) => indiceAtual !== index)`. `filter` devolve somente os elementos para os quais a condição é verdadeira. O `_` é um nome escolhido para indicar que o valor daquele item não é usado; o código só precisa do índice.

Remover essa linha não faz um `DELETE` HTTP. A associação no banco só será alterada quando o usuário salvar a pizza. Se cancelar o modal, essas alterações de rascunho não são enviadas.

**42. Validar pizza reúne regras do formulário antes de iniciar HTTP.**

`validarPizza()` lê o estado `novaPizza`, cria um objeto `erros` e converte o preço com `converterMoedaParaNumero`.

Ela exige nome não vazio, tamanho escolhido, preço finito e positivo, pelo menos uma linha de composição, ingrediente escolhido em cada linha e quantidade finita e positiva em cada linha.

O `for (const item of novaPizza.ingredientes)` percorre os valores do array. É a versão JavaScript de um laço sobre cada item; não percorre os IDs automaticamente.

```tsx
const ids = novaPizza.ingredientes.map(item => item.ingrediente_id)
if (new Set(ids).size !== ids.length) {
  erros.ingredientes = 'O mesmo ingrediente não pode ser adicionado duas vezes.'
}
```

`map` extrai os IDs. `new Set(ids)` instancia uma coleção que mantém valores únicos. `.size` conta quantos valores distintos restaram; `ids.length` conta quantos IDs havia. Se os números forem diferentes, existia repetição.

Aqui `new Set(...)` realmente instancia um objeto JavaScript. Isso é diferente de `type Pizza`, que apenas descreve um tipo para o compilador.

Mensagens podem ser sobrescritas: mais de uma condição escreve em `erros.ingredientes`. Por exemplo, duas linhas sem seleção têm dois IDs vazios iguais, o que pode substituir “Selecione todos os ingredientes” pela mensagem de duplicidade.

Ao terminar, a função chama `setErrosFormulario(erros)` e devolve se o objeto está vazio. Não valida o banco, não confere o nome contra todas as pizzas armazenadas e não garante que o status de um ingrediente ainda seja o mesmo no servidor. Essas regras continuam dependendo do backend.

A duplicidade de ingrediente é impedida nesta função, mas não há regra correspondente no serializer/model recebido. Essa diferença importa quando a API é chamada fora da sua tela.

**43. A mesma função `cadastrarPizza` decide entre POST e PUT.**

Apesar do nome, `cadastrarPizza` atende cadastro e edição. O evento de envio chega pelo `onSubmit={cadastrarPizza}`. A função chama `event.preventDefault()` e retorna cedo se `validarPizza()` falhar.

O objeto `dados` contém nome normalizado, preço convertido, tamanho e uma composição mapeada para somente ID e quantidade. Os dados descritivos que vêm na leitura — como nome do ingrediente e custo médio — não precisam ser enviados novamente.

```tsx
const request = modoEdicao && pizzaSelecionada
  ? axios.put<Pizza>(
      `http://localhost:8000/api/pizzas/${pizzaSelecionada.id}/`,
      dados,
    )
  : axios.post<Pizza>('http://localhost:8000/api/pizzas/', dados)
```

O `&&` verifica se a tela está em modo de edição e tem uma pizza selecionada. O ternário escolhe uma das duas chamadas. A variável `request` armazena a Promise resultante; ela não é o `request` do Python. A requisição já é iniciada quando `axios.put` ou `axios.post` é chamado.

`<Pizza>` informa o tipo esperado da resposta. O primeiro argumento é a URL, o segundo é o corpo. O `PUT` inclui o ID porque atualiza um registro existente; o `POST` utiliza o endereço da coleção porque cria outro registro.

No `.then`, o modo de edição substitui a linha por ID com `map`; o modo de cadastro acrescenta o retorno ao array com spread. Depois, o código limpa formulário, seleção, modo de edição, visibilidade do modal e erros.

No `.catch`, o detalhe do backend é registrado no console, enquanto `setErro('Não foi possível salvar a pizza.')` apresenta uma mensagem genérica. O modal permanece aberto, mas essa mensagem geral está na área principal da página, fora do `Dialog`. Não existe, nessa função, um mapeamento das mensagens do serializer para `errosFormulario`.

Esse é o ponto em que um erro como nome duplicado ou ingrediente inativo poderia ser transformado em mensagem ao lado do campo apropriado, se você evoluir o tratamento.

**44. Abrir o formulário prepara o estado; não faz outra consulta individual.**

```tsx
const abrirFormularioPizza = (pizza?: Pizza) => {
  if (pizza) {
    // Preenche a edição.
  } else {
    // Prepara o cadastro.
  }
  // Limpa erros e abre o modal.
}
```

`pizza?` é um parâmetro opcional. O botão de adicionar chama `abrirFormularioPizza()` sem argumento. O botão de editar chama `abrirFormularioPizza(pizza)` com o objeto da linha.

Na edição, a função armazena a pizza selecionada, ativa `modoEdicao` e monta o formulário. Usa `String` no preço e nas quantidades porque esses campos do formulário são textuais. Mapeia as associações para os campos de edição, descartando do rascunho informações somente de exibição.

No cadastro, remove a seleção, desativa o modo de edição e usa `formularioInicial`. Nos dois casos limpa mensagens e abre o diálogo.

Essa função usa os dados que já estão em `pizzas`. Não faz `GET /pizzas/{id}/`. A rota de consulta individual existe no backend, mas não é chamada por esse botão.

Usar `onClick={() => abrirFormularioPizza()}` também impede que o evento de clique seja passado inadvertidamente como se fosse o parâmetro `pizza`. No botão de editar, a função intermediária fornece explicitamente a pizza correta.

O fechamento do diálogo e o botão cancelar redefinem visibilidade, modo, seleção e formulário. Não enviam as mudanças ao servidor.

**45. O toggle de pizzas atualiza somente o status e a linha retornada.**

`alternarStatusPizza(pizza)` faz `POST` para `/api/pizzas/${pizza.id}/toggle_ativo/`. Nesse caso, o código não fornece um corpo de confirmação. A ação de backend também não exige confirmação.

No sucesso, a lista é reconstruída substituindo a pizza daquele ID pelo objeto da resposta. No erro, o detalhe vai ao console e a mensagem geral vai ao estado `erro`.

`acoesDaLinha` produz o botão com ícone, cor, acessibilidade e tooltip baseados em `pizza.ativo`. O botão X significa “desativar”; o check significa “ativar”. O fato de o ícone ser X não muda o verbo HTTP para `DELETE`.

Depois da resposta, o React renderiza o novo status e, por consequência, o botão passa a apresentar a ação inversa. O backend decide o novo valor; o frontend utiliza o valor retornado.

**46. A tabela de pizzas contém expansão, formatação e indicação de ingrediente inativo.**

`value={pizzas}` fornece a coleção. `dataKey="id"` indica a identidade. `emptyMessage` fornece o texto quando não há registros.

`expandedRows={pizzasExpandidas}` controla o que está expandido. `onRowToggle` recebe o evento e utiliza `e.data` para atualizar esse estado. O `as Pizza[]` é uma afirmação de tipo, sem conversão em tempo de execução. O código espera uma representação em array; o formato usado precisa corresponder ao contrato da versão do PrimeReact utilizada.

`<Column expander ... />` fornece o controle de expansão. `rowExpansionTemplate` é uma função que recebe a pizza e descreve o conteúdo expandido.

Se a pizza tiver composição, `map` transforma cada item em `<li>` com nome, quantidade e unidade. Se não tiver, a expressão ternária mostra o texto de ausência.

A propriedade `key` ajuda o React a identificar elementos entre renderizações. Na lista expandida, seu código combina ID da pizza, ID do ingrediente e índice. No formulário, usa somente o índice. Usar índices em uma lista que permite remoções pode dificultar a preservação da identidade visual de cada linha; um identificador local estável é uma possibilidade de melhoria. [Listas e chaves no React](https://react.dev/learn/rendering-lists).

As colunas simples leem ID, nome e tamanho. O tamanho é apresentado pelo código do campo, como `G`, pois a coluna não faz conversão para o rótulo “Grande”. As colunas de preço e custo chamam `moeda(...)` pelo `body`.

Na coluna de status, `pizza.ativo ? 'Ativa' : 'Inativa'` escolhe texto; outra expressão escolhe cor. Depois:

```tsx
pizza.ingredientes.some(
  ingrediente => ingrediente.ingrediente_ativo === false
)
```

`some` devolve `true` quando ao menos um item satisfaz a condição. Seu código verifica se existe ingrediente inativo na composição e, se existir, mostra “Ingrediente inativo vinculado à pizza”.

O aviso não desativa a pizza automaticamente, não bloqueia o botão editar e não corrige o serializer. Ele é uma indicação visual baseada nos dados que a tela já recebeu.

O botão da coluna editar abre o modal com a pizza da linha. A coluna de status passa `acoesDaLinha` como função de renderização. `headerStyle` e `bodyStyle` definem medidas e alinhamento dessas colunas.

**47. O modal de pizzas usa uma composição dinâmica de selects e inputs.**

O título do diálogo é escolhido por `modoEdicao ? 'Editar pizza' : 'Nova pizza'`. `visible` é controlado por `popupAberto`. O formulário usa `onSubmit={cadastrarPizza}` e os botões finais alteram o texto conforme o modo.

O nome e o tamanho são campos controlados por `novaPizza`. O select de tamanho oferece as opções `P`, `M` e `G`, compatíveis com o model. O preço usa `handleMoedaChange`.

O input de preço tem `type="text"`. Nesse tipo, os atributos `min="0.01"` e `step="0.01"` não implementam a validação numérica de um input `number`. A máscara, a validação JavaScript e o backend são os mecanismos efetivos relevantes.

`novaPizza.ingredientes.map((item, index) => (...))` produz uma linha visual para cada item do rascunho. Cada linha contém select de ingrediente, input de quantidade, possíveis erros, unidade e botão de remoção.

O select recebe `value={item.ingrediente_id}`. Quando muda, chama:

```tsx
alterarIngrediente(index, 'ingrediente_id', event.target.value)
```

As opções são construídas por:

```tsx
ingredientesDisponiveis
  .filter(ingrediente => ingrediente.ativo)
  .map(ingrediente => (
    <option key={ingrediente.id} value={ingrediente.id}>
      {ingrediente.nome}
    </option>
  ))
```

O `filter` mantém apenas ativos; o `map` transforma cada objeto restante em opção HTML. O ID é o valor; o nome é o texto exibido.

Essa versão remove todos os ingredientes inativos das opções, inclusive o que estava selecionado em uma composição antiga. O estado pode continuar guardando seu ID, mas o select não tem a opção correspondente. O backend também rejeita esse ID no envio da composição. Por isso, permitir manter ingredientes inativos existentes exige um ajuste coordenado na tela e no serializer.

Para a quantidade, o código usa `type="number"`, mínimo `0.001` e passo `0.001`. O `onChange` limpa valores negativos e chama `alterarIngrediente(index, 'quantidade', value)`. A validação posterior exige quantidade positiva.

A unidade é obtida com `ingredientesDisponiveis.find(...)`. `find` devolve o primeiro objeto cujo ID corresponde ao selecionado, ou `undefined`. `?.unidade_De_Medida` acessa a unidade somente se esse objeto existir. Nenhuma consulta HTTP é feita para isso: a busca é no array já carregado.

Os erros de composição e quantidade são mostrados somente quando `index === 0`. Isso significa que a mensagem é geral, posicionada na primeira linha, e não uma mensagem individual por linha. Se o array estiver vazio, não haverá primeira linha: a validação registra “Adicione pelo menos um ingrediente”, mas o trecho que exibiria essa mensagem não é renderizado. Esse é um ponto concreto a ajustar no posicionamento da mensagem.

O botão de lixeira tem `type="button"` para não enviar o formulário. Ele remove uma linha do rascunho. O botão “Adicionar ingrediente” também é `type="button"`. O botão final de salvar/cadastrar é `type="submit"`, que conduz ao fluxo HTTP.

**48. Estas são todas as chamadas HTTP presentes nas duas telas.**

| Tela e função | Chamada | Método no backend |
|---|---|---|
| Ingredientes: `carregarIngredientes` | `GET /api/ingredientes/` | `list` herdado |
| Ingredientes: `cadastrarIngrediente` | `POST /api/ingredientes/` | `create` herdado, com `perform_create` personalizado |
| Ingredientes: `salvarEdicao` | `PUT /api/ingredientes/{id}/` | `update` herdado, com `perform_update` personalizado |
| Ingredientes: `alternarStatusIngrediente` | `POST /api/ingredientes/{id}/toggle_ativo/` | `toggle_ativo` escrito por você |
| Pizzas: efeito inicial | `GET /api/pizzas/` | `list` herdado |
| Pizzas: efeito inicial | `GET /api/ingredientes/` | `list` herdado do outro app |
| Pizzas: `cadastrarPizza`, modo novo | `POST /api/pizzas/` | `create` herdado; serializer com criação personalizada |
| Pizzas: `cadastrarPizza`, modo edição | `PUT /api/pizzas/{id}/` | `update` herdado; serializer com atualização personalizada |
| Pizzas: `alternarStatusPizza` | `POST /api/pizzas/{id}/toggle_ativo/` | `toggle_ativo` escrito por você |

Não há chamadas `PATCH`, `DELETE`, `GET` individual ou `adicionar_estoque` nesses dois componentes. Essas operações podem existir como rotas do backend sem aparecer como recursos de interface nessas telas.

O Axios oferece métodos como `get`, `post`, `put`, `patch` e `delete`; nos métodos com corpo usados aqui, a URL vem primeiro e os dados vêm depois. A Promise permite registrar os tratamentos de sucesso e falha. [Referência da API do Axios](https://axios.rest/pages/advanced/api-reference).

`localhost` designa a própria máquina em que o navegador está sendo executado. Em desenvolvimento isso pode apontar ao Django local. Se a página fosse publicada e aberta em outro computador, esse endereço não apontaria automaticamente ao seu servidor original. O endereço-base da API é uma configuração que normalmente se centraliza ao preparar outros ambientes.

Se frontend e backend estiverem em origens diferentes, as configurações de CORS do backend podem participar da permissão de leitura pelo navegador. Nada nesses componentes substitui essa configuração; ela não pode ser conferida sem os arquivos correspondentes.

**49. Onde criar um método novo depende de quem precisa executá-lo.**

| Necessidade | Lugar adequado no seu desenho atual | Como seria chamado |
|---|---|---|
| Formatar moeda visualmente | Função utilitária no frontend | `body`, input ou handler |
| Abrir/fechar modal | Função do componente | Evento de clique/fechamento |
| Impedir envio de formulário vazio | Validação frontend | Função de submit |
| Validar um campo da API | `validate_nome_do_campo` no serializer | `serializer.is_valid()` |
| Comparar dois campos enviados | `validate(self, attrs)` | `serializer.is_valid()` |
| Restringir os objetos aceitos em uma relação | Campo relacional/validação do serializer | Validação da entrada |
| Definir consultas disponíveis à view | `get_queryset` | Listagem e busca individual |
| Acrescentar dados decididos no servidor ao salvar | `perform_create` ou `perform_update` | Ações herdadas |
| Gravar uma estrutura aninhada | `create`/`update` do serializer | `serializer.save()` |
| Mudar a política de exclusão | `perform_destroy` | `destroy` herdado |
| Expor uma operação HTTP adicional | Método do ViewSet com `@action` | Rota adicional do router |
| Reutilizar uma regra da entidade fora da API | Método do model ou função de domínio | Chamada explícita pelos fluxos necessários |
| Garantir integridade em todos os caminhos de gravação | Restrição apropriada do banco/model | Aplicada na persistência |
| Controlar quem pode executar ações | Classes/configuração de permissões | Ciclo de atendimento do DRF |

Uma função de cálculo pode morar no model e ser chamada pela view ou pelo serializer. Ela não passa a rodar sozinha por estar no model: é preciso definir quem a chama. Se optar por sobrescrever `save`, deve compreender quais caminhos passam por ele, quais operações o contornam e como evitar recursão.

Não é obrigatório criar uma camada service para cada operação. No estágio atual, você pode organizar pequenas regras em methods do model e orquestrá-las na view/serializer. Uma operação que envolve várias entidades e precisa ser reutilizada pode justificar uma função de serviço mais adiante.

Não é adequado colocar somente no React uma regra que protege o banco, porque a API pode ser chamada sem utilizar sua tela. Ao mesmo tempo, manter a validação visual é útil para avisar cedo e posicionar mensagens junto aos campos.

**50. Exemplos de extensão mostram o ponto exato de entrada.**

Para uma regra simples de campo, você criaria um método no serializer correspondente. Exemplo didático: se fosse decidido que o estoque mínimo deve ter uma regra adicional, ela entraria em `validate_estoque_minimo`. A função precisa devolver o valor aprovado ou levantar `ValidationError`.

Para uma regra que compara quantidade e custo, `validate(self, attrs)` é apropriado porque recebe o conjunto. Na edição parcial, seria necessário combinar os valores enviados com os existentes em `self.instance`, respeitando a diferença entre chave omitida e valor nulo.

Para impedir ingredientes repetidos na composição pelo backend, `PizzasSerializer.validate_ingredientes` pode verificar os IDs dos objetos já validados. Uma restrição única para `(pizza, ingrediente)` também protegeria a tabela associativa. A tela já possui sua checagem com `Set`, mas não substitui essas proteções.

Para aceitar ingrediente inativo somente quando ele já pertence à pizza editada, a validação precisa conhecer a pizza e seus vínculos atuais. O serializer externo possui `self.instance`; não se deve assumir que cada serializer filho aninhado recebeu automaticamente a associação existente em `self.instance`. Também é preciso deixar o campo relacional alcançar o objeto para que a validação contextual possa decidir; o filtro atual `ativo=True` rejeita antes dessa distinção.

No frontend, a regra correspondente estaria no `filter` das opções de **cada linha**: aceitar ingredientes ativos e o ingrediente inativo que permanece selecionado naquela linha. Se a pessoa remover ou substituir a seleção, aquele inativo deixa de ser oferecido ali. Se uma remoção já foi salva, o backend deve rejeitar uma tentativa posterior de adicioná-lo como novo vínculo.

Para organizar a entrada de estoque, um serializer específico de entrada poderia declarar `quantidade_entrada` e `custo_total_entrada`, seus tipos e limites. A ação instanciaria esse serializer com `data=request.data`, chamaria `is_valid(raise_exception=True)` e usaria `validated_data`. Isso faria essa operação ter validação explícita de entrada, em vez de conversões diretas sem tratamento.

Para expor uma consulta adicional de estoque baixo, um exemplo didático de esqueleto seria:

```python
# Exemplo de localização e estrutura; não está implementado no seu projeto.
from django.db.models import F

@action(detail=False, methods=['get'])
def estoque_baixo(self, request):
    ingredientes = self.get_queryset().filter(
        quantidade_estoque__lte=F('estoque_minimo')
    )
    serializer = self.get_serializer(ingredientes, many=True)
    return Response(serializer.data)
```

O método ficaria dentro de `IngredientesViewSet`; o import ficaria no início do arquivo. `detail=False` indica uma consulta da coleção. `__lte` significa menor ou igual. `F('estoque_minimo')` compara com outra coluna do próprio registro, e não com uma string literal. A rota seria gerada pelo router usando o nome da ação. O frontend precisaria de uma chamada `axios.get` e de um local para exibir o resultado.

Esse exemplo retorna dados; não envia notificações automaticamente nem cria um agendamento. Esses seriam outros comportamentos a implementar se fossem desejados.

**51. Uma simulação completa ajuda a distinguir memória, HTTP e banco.**

Considere a criação de uma pizza com 0,200 KG de mussarela e preço de R$ 45,00.

1. Você abre o modal. `popupAberto` passa a verdadeiro; o banco não muda.
2. Digita o nome. `handleInputChange` atualiza `novaPizza.nome`; o banco não muda.
3. Digita o preço. A máscara guarda texto como `'R$ 45,00'`; o banco não muda.
4. Adiciona uma linha. `adicionarIngrediente` acrescenta um item ao array local.
5. Seleciona o ingrediente 7. `alterarIngrediente` guarda o ID numérico 7 na linha.
6. Digita a quantidade. O rascunho guarda a string da entrada.
7. Clica em cadastrar. O formulário chama `cadastrarPizza`.
8. A validação local verifica os campos; se reprovar, nenhuma requisição é iniciada por esse caminho.
9. O payload transforma preço e quantidade em números e envia o ID do ingrediente.
10. `axios.post` envia HTTP ao backend.
11. O router seleciona a ação herdada `create` de `PizzasViewSet`.
12. A ação instancia `PizzasSerializer` e valida a entrada aninhada.
13. O campo relacional procura o ingrediente 7 ativo e fornece seu objeto.
14. O `perform_create` chama `serializer.save()`.
15. O serializer escolhe seu `create` personalizado.
16. O model da pizza é criado no banco.
17. A associação `PizzaIngrediente` é criada apontando ao ID da pizza e ao ID 7.
18. O custo da receita é calculado e armazenado; a view ainda o recalcula novamente.
19. A representação da pizza é devolvida com status 201.
20. O Axios conclui a Promise e executa o `.then`.
21. A lista `pizzas` recebe o objeto retornado; o formulário é limpo e o modal fecha.
22. O React renderiza a tabela contendo o novo registro.

Na edição, as diferenças centrais são: existe um ID-alvo; usa-se `PUT`; o serializer recebe `instance`; `.save()` escolhe `update`; a pizza mantém sua identidade; a composição enviada é substituída; a tela troca a linha daquele ID em vez de acrescentar outra.

No toggle, o caminho é menor: clique, POST da ação, `get_object`, mudança de booleano, `model.save`, serializer somente de saída, resposta e substituição da linha. Não passa pelos mesmos passos de validação do cadastro.

**52. Alguns problemas do código explicam comportamentos que podem parecer inesperados.**

Estes pontos são conclusões da leitura dos arquivos, não resultados de testes com o projeto em execução.

| Ponto encontrado | Efeito provável ou direto | Onde tratar |
|---|---|---|
| `FormIngrediente` mantém `custoMedio` e `ativo` obrigatórios | Objetos de formulário apresentados não satisfazem o tipo | Tipo do formulário |
| `perform_update` de ingredientes exige duas chaves por colchetes | PATCH parcial pode gerar `KeyError` | Gancho e validação de edição |
| Serializer aceita custo nulo, model não | Algumas atualizações podem falhar na persistência | Normalização e contrato do serializer |
| Entrada de estoque não valida por serializer | Dados inválidos podem causar exceções ou saldos inadequados | Serializer específico e ação |
| `get_queryset` de pizzas salva custos | Leitura e ações individuais geram gravações e recálculos de todas as pizzas | Local do cálculo/consulta |
| Custo calculado no serializer e na view | Trabalho e gravações repetidos | Centralização da regra |
| Substituição da composição sem transação explícita | Falha intermediária pode deixar persistência parcial se não houver transação externa | Operação de gravação |
| Ingredientes inativos rejeitados pelo campo relacional | PUT de composição antiga pode falhar | Validação contextual da composição |
| Select mostra somente ativos | Inativo selecionado perde opção visível | Filtro por linha do formulário |
| Duplicidade de ingrediente validada apenas no frontend | Chamadas diretas podem repetir associações | Serializer e restrição de unicidade |
| `DELETE` de pizzas permanece padrão | Exclusão física continua disponível na API | Política de exclusão da view/model |
| Erros de pizza do servidor viram mensagem geral fora do modal | Usuário não recebe detalhe junto ao campo | `.catch` e posicionamento visual |
| Erro de composição mostrado só na primeira linha | Com zero linhas, mensagem pode não aparecer | JSX fora do `map` |
| Validação inline escreve em erros do modal de cadastro | Falha pode não ser visível na linha | Estado e apresentação de erros inline |
| Quantidade decimal de ingrediente sem `step` no formulário | Validação nativa pode restringir frações | Propriedades dos inputs |

Há ainda uma diferença entre a especificação textual do DER anexada e o código atual: a especificação descreve entidades e proteções adicionais, enquanto estes models implementam ingredientes, pizzas e sua associação. Por isso, esta explicação usa o código como fonte do comportamento em execução; não presume que cada regra descrita na especificação já esteja implementada.

**53. Um pequeno glossário de operações ajuda a reler os arquivos.**

| Expressão | Leitura prática |
|---|---|
| `objects.all()` | Construir consulta de todos os registros |
| `.filter(...)` | Restringir aos que atendem às condições |
| `.exclude(...)` | Retirar os que atendem à condição |
| `.exists()` | Verificar se existe algum resultado |
| `.order_by('id')` | Ordenar pelo ID crescente |
| `.select_related('ingrediente')` | Trazer a relação individual na consulta |
| `.get('campo', padrão)` | Ler uma chave com valor alternativo |
| `dicionario['campo']` | Ler uma chave que precisa existir |
| `.pop('campo', padrão)` | Retirar e devolver uma chave |
| `.items()` | Percorrer pares chave/valor |
| `setattr(objeto, nome, valor)` | Atribuir atributo cujo nome está em uma variável |
| `**dados` | Passar chaves do dicionário como argumentos nomeados |
| `.save(update_fields=[...])` | Persistir somente os campos indicados do model |
| `.delete()` | Solicitar exclusão física pelo ORM |
| `.map(...)` | Criar outro array transformando cada item |
| `.filter(...)` no JavaScript | Criar outro array mantendo itens aprovados |
| `.find(...)` | Encontrar o primeiro item aprovado |
| `.some(...)` | Verificar se algum item é aprovado |
| `...objeto` / `...array` | Espalhar propriedades/elementos em outra estrutura |
| `condição ? A : B` | Escolher um valor no JavaScript |
| `A if condição else B` | Escolher um valor no Python |
| `?.` | Acesso ou chamada opcional no JavaScript |
| `??` | Valor alternativo quando o anterior é nulo/indefinido |
| `===` | Comparação estrita no JavaScript |
| `is True` | Exigir o objeto booleano verdadeiro no Python |
| `return` | Devolver um resultado e encerrar aquela função |
| `raise` | Interromper o fluxo normal com uma exceção |

**54. O código abaixo torna visível o CRUD que você herdou.**

Os trechos desta parte são **pseudocódigo Python simplificado**, escrito para explicar as chamadas. Não são uma reprodução integral do código-fonte de sua versão instalada e não devem ser colados no projeto. Detalhes de headers, caches, filtros, paginação e tratamento do framework foram reduzidos quando não eram o foco.

O cadastro herdado pode ser acompanhado assim:

```python
# Esquema didático da ação HTTP na view.
def create(self, request, *args, **kwargs):
    serializer = self.get_serializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    self.perform_create(serializer)
    return Response(serializer.data, status=201)

# Esquema do gancho padrão, antes de sua personalização.
def perform_create(self, serializer):
    serializer.save()
```

`*args` permite receber argumentos posicionais adicionais; `**kwargs` permite receber argumentos nomeados adicionais em um dicionário. O framework usa assinaturas flexíveis para os dados associados ao atendimento de uma rota. Seu método `perform_create` não precisa recebê-los porque a ação o chama apenas com o serializer.

Observe `self.perform_create(serializer)`: a ação é herdada, mas a busca desse método acontece no objeto da sua classe. Por isso o seu cálculo de custo é executado nessa linha. Não é necessário reescrever `create` inteiro para acrescentar o cálculo.

O núcleo conceitual de `serializer.save()` é:

```python
# Esquema didático: a validação precisa ter sido bem-sucedida antes.
def save(self, **valores_definidos_pelo_servidor):
    dados = {
        **self.validated_data,
        **valores_definidos_pelo_servidor,
    }

    if self.instance is not None:
        self.instance = self.update(self.instance, dados)
    else:
        self.instance = self.create(dados)

    return self.instance
```

O critério fundamental é a presença de `self.instance`. A chamada `save(custoMedio=30)` não significa executar um método chamado `custoMedio`: acrescenta um argumento de gravação. O valor decidido pelo servidor entra em `dados` antes da criação/atualização.

A criação simples herdada do `ModelSerializer`, aplicada ao seu ingrediente, pode ser entendida como:

```python
# Esquema restrito ao seu model de ingredientes, sem relações aninhadas.
def create(self, validated_data):
    ingrediente = Ingredientes.objects.create(**validated_data)
    return ingrediente
```

O framework real é genérico: usa o model configurado em `Meta` e trata outros detalhes. Você não vê esse trecho escrito em `IngredientesSerializer` porque ele aproveita a implementação da classe base. Já para pizzas você escreveu uma implementação própria, que também cria as associações.

A consulta individual e a listagem podem ser compreendidas assim:

```python
# Esquema sem detalhar filtros/paginação.
def retrieve(self, request, *args, **kwargs):
    instance = self.get_object()
    serializer = self.get_serializer(instance)
    return Response(serializer.data)

def list(self, request, *args, **kwargs):
    queryset = self.get_queryset()
    serializer = self.get_serializer(queryset, many=True)
    return Response(serializer.data)
```

Não há `.save()` nem `.is_valid()` nesses esquemas de leitura. No seu projeto, entretanto, a personalização de `PizzasViewSet.get_queryset` introduz gravações de custo durante a obtenção da coleção. Assim, o comportamento final depende tanto da ação herdada quanto do método que ela chama.

A atualização pode ser acompanhada pelo seguinte esquema:

```python
def update(self, request, *args, **kwargs):
    partial = kwargs.pop('partial', False)
    instance = self.get_object()
    serializer = self.get_serializer(
        instance,
        data=request.data,
        partial=partial,
    )
    serializer.is_valid(raise_exception=True)
    self.perform_update(serializer)
    return Response(serializer.data)

def partial_update(self, request, *args, **kwargs):
    kwargs['partial'] = True
    return self.update(request, *args, **kwargs)

def perform_update(self, serializer):
    serializer.save()
```

Na construção do serializer, o primeiro argumento posicional `instance` identifica o objeto existente; `data=` identifica a entrada. São os mesmos conceitos que poderiam ser escritos com `instance=instance` explicitamente.

O `PATCH` não exige um segundo algoritmo inteiramente independente de atualização. A ação parcial altera a configuração e utiliza o caminho de `update`. A seguir, seu `perform_update` personalizado continua sendo chamado; é exatamente por isso que o acesso obrigatório a chaves ausentes pode quebrar seu PATCH de ingredientes.

A atualização simples de ingredientes, já dentro do serializer, segue esta ideia:

```python
def update(self, instance, validated_data):
    for nome_do_atributo, valor in validated_data.items():
        setattr(instance, nome_do_atributo, valor)
    instance.save()
    return instance
```

Aqui o `save` é do **model**, porque `instance` é um ingrediente. No gancho da view, `serializer.save()` era o método do **serializer**. Os nomes iguais não tornam as duas chamadas equivalentes.

Por fim, o esquema de exclusão:

```python
def destroy(self, request, *args, **kwargs):
    instance = self.get_object()
    self.perform_destroy(instance)
    return Response(status=204)

def perform_destroy(self, instance):
    instance.delete()
```

No ingrediente, sua sobrescrita substitui o segundo método pela verificação de vínculo e por `instance.save(update_fields=['ativo'])`. Na pizza, o segundo método continua herdado e chega a `instance.delete()`.

Você não deve confundir o retorno de um gancho com a resposta HTTP. Seus `perform_create` e `perform_update` não retornam uma `Response`, pois quem continua o trabalho e constrói essa resposta é a ação que os chamou. Por outro lado, a ação adicional `toggle_ativo` é responsável por devolver sua própria `Response`.

Para estudar acompanhando o editor, comece pelo `POST` de ingredientes e siga as chamadas da parte 15 usando estes esquemas. Depois percorra o `PUT` da parte 17 e compare com a criação/atualização aninhada de pizzas nas partes 21 e 22. As partes 26 a 48 explicam cada grupo de funções das duas telas e mostram onde os eventos visuais entram nesses mesmos fluxos.
