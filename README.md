# 🍔 Cantina CESAR: o intervalo rende

> Hackathon Relâmpago #1 · FDS 2026.2 · Cap. 5 (Princípios de Projeto) · **Entrega 1: versão go-horse**

## O que é

Temos 15 minutos de intervalo e metade vai embora na fila. Com o app, o aluno **faz o pedido antes**
(durante a aula), paga, recebe uma **senha** com a **estimativa de quando o lanche fica pronto**, e é
**avisado** quando pode retirar. A cozinha vê a fila em ordem e avança cada pedido
(recebido → em preparo → pronto → retirado). Resultado: chegou, pegou, comeu.

Funcionalidades da PoC: cardápio do dia com itens e **combos**, pedido antecipado, pagamento
(Pix simulado e carteira pré-paga), senha de retirada, estimativa de fila e aviso de "pedido pronto".

## Como rodar

Requer Python 3.10+. Nenhuma dependência para rodar o app.

```bash
python -m cantina            # menu interativo (aluno e cozinha)
python -m cantina --demo     # roteiro automático (usado no vídeo)

pip install -r requirements-dev.txt
python -m pytest -q          # 10 testes
```

## Equipe

| Integrante | Papel |
|---|---|
| _Nome 1_ | _ex.: domínio (Pedido, Combo) e testes_ |
| _Nome 2_ | _ex.: serviço e adaptadores_ |
| _Nome 3_ | _ex.: interface de terminal e vídeo_ |
| _Nome 4_ | _ex.: README e métricas_ |

## Decisões de stack

- **Python puro, sem framework:** o foco é a estrutura do código, não a infraestrutura. Roda em qualquer máquina sem instalar nada.
- **Interface de terminal:** suficiente para provar o fluxo. Como as telas dependem só das interfaces `VisaoCliente`/`VisaoCozinha`, uma interface web pode substituí-la sem mexer no domínio.
- **Repositório em memória:** o enunciado dispensa banco. Trocar por SQLite é escrever uma nova implementação de `RepositorioDePedidos`.
- **Dinheiro em centavos (`int`):** evita erros de arredondamento de `float`.
- **IA usada para gerar o código-base;** a equipe revisou, entendeu e é responsável por cada decisão.

## Arquitetura

```
cantina/
├── dominio/          regras de negócio puras (não dependem de nada externo)
│   ├── produto.py      Produto (contrato), Item, Combo
│   ├── pedido.py       Pedido e seu ciclo de vida
│   └── cardapio.py
├── portas.py         interfaces que o núcleo exige: Repositório, Pagamento, Notificador
├── visoes.py         interfaces que o núcleo oferece: VisaoCliente, VisaoCozinha
├── servicos/         casos de uso: ServicoDePedidos, EstimadorDeFila
├── adaptadores/      implementações concretas (memória, Pix, carteira, console)
├── app.py            raiz de composição: único lugar que dá "new" nas classes concretas
└── __main__.py       interface de terminal
```

As dependências apontam **para dentro**: adaptadores e interface conhecem o domínio; o domínio não conhece nenhum deles.

## 🎯 Onde estão os princípios

| Princípio | Onde | Por que deixa o projeto melhor |
|---|---|---|
| **Integridade conceitual** | Todo o projeto | Um vocabulário único (Pedido, Senha, Cardápio, Combo, Fila) igual no código, nas telas e neste README. Todo pedido passa pelo mesmo ciclo de vida, sem atalhos. |
| **Ocultamento de informação** | `dominio/pedido.py:20` (`_TRANSICOES`) e `:74` (`_mudar_para`) | O status é privado e só muda por métodos que validam a transição. É impossível "preparar" um pedido não pago. Se a regra mudar (ex.: cancelamento), muda num lugar só. |
| **Coesão / Responsabilidade Única** | `servicos/estimador.py:7`, `dominio/cardapio.py`, adaptadores | Cada classe tem um motivo para mudar: trocar a fórmula da estimativa não toca em pagamento nem em notificação. |
| **Acoplamento / Inversão de Dependências** | `servicos/servico_pedidos.py:23` (construtor) e `app.py:29` | O serviço recebe abstrações (`portas.py`) por injeção. Por isso os testes usam um notificador fake e, no futuro, trocamos memória por banco ou console por WhatsApp sem mexer no serviço. |
| **Aberto/Fechado** | `adaptadores/pagamentos.py` | Adicionar um meio de pagamento é criar uma classe e registrá-la em `app.py`. `ServicoDePedidos` não é alterado. |
| **Substituição de Liskov** | `portas.py` (contrato de `cobrar`) e `tests/test_servico.py:35` | `CarteiraPrePaga` sem saldo **retorna `False`** como diz o contrato, em vez de lançar uma exceção inesperada. O mesmo teste roda com qualquer meio de pagamento. |
| **Segregação de Interfaces** | `visoes.py:13` e `:28`; uso em `__main__.py:15` e `:40` | A tela do aluno recebe uma `VisaoCliente` e não enxerga `avancar()`; a da cozinha recebe `VisaoCozinha` e não enxerga pagamento. |
| **Composição em vez de herança** | `dominio/produto.py:33` (`Combo`) | Um Combo **contém** produtos e calcula preço e tempo a partir deles. Não herda de `Item`, então pode até conter outro Combo. Pedido aceita os dois pelo contrato `Produto`. |
| **Demeter** | `dominio/pedido.py:49` (`total`) | O serviço pergunta `pedido.total()` em vez de percorrer `pedido._itens[i][0].preco()`. A estrutura interna do pedido pode mudar sem quebrar quem o usa. |
| **Métricas** | Seção abaixo | Medimos para ter evidência, não opinião. |

## 📏 Métricas

Geradas com `radon` (LOC e complexidade ciclomática) e `cohesion` (coesão de classe, análoga ao inverso do LCOM).

```bash
radon raw cantina -s     # LOC
radon cc cantina -a -s   # complexidade ciclomática
cohesion -d cantina      # coesão por classe
```

| Métrica | Resultado |
|---|---|
| LOC total (pacote `cantina`) | 548 (382 SLOC) |
| Maior arquivo | `servicos/servico_pedidos.py`, 102 LOC |
| Complexidade ciclomática média | **1,66** (nota A) em 90 blocos; nenhuma função acima de 5 |
| Coesão `Cardapio` | 100% |
| Coesão `Item` / `Combo` | 66,7% |
| Coesão `Pedido` | 30,6% (ver abaixo) |
| Acoplamento (CBO, contado à mão) | `ServicoDePedidos` depende de 5 abstrações e 0 classes concretas de infraestrutura |

**Leitura crítica:** a coesão de `Pedido` é a mais baixa porque ele mistura cálculo (total, tempo) com
ciclo de vida (status). É uma candidata a refatoração na Entrega 2 (por exemplo, extrair a máquina de
estados para uma classe própria).

## Próximos passos (Entrega 2)

Interface web, persistência em SQLite, cancelamento de pedido, horário de retirada agendado e notificação real.
