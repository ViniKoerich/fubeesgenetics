# FU Bees Genetics — Genetic Queen Filter
## Documento Mestre de Implementação (MVP)

**Status:** especificação funcional e técnica consolidada  
**Data de consolidação:** 2026-10-06  
**Runtime de referência:** OpenStarbound  
**Dependência de conteúdo:** Frackin' Universe (FU)  
**Namespace do addon:** `FUBeesGenetics` / `fubeesgenetics_*`

---

# 0. Como usar este documento

Este documento é deliberadamente autossuficiente. Ele deve ser suficiente para que um desenvolvedor, outro chat ou agente de implementação comece o projeto sem conhecer nenhuma conversa anterior.

Para cada decisão importante, este documento especifica:

- **o que** deve ser implementado;
- **como** deve funcionar;
- **por que** essa solução foi escolhida;
- **quais arquivos/APIs do FU ou OpenStarbound servem de referência**;
- **quais alternativas foram descartadas**;
- **quais invariantes não podem ser quebradas**;
- **quais edge cases precisam ser tratados**;
- **como validar o resultado**.

Quando houver diferença entre uma decisão funcional obrigatória e um detalhe que pode ser ajustado durante implementação/testes, isso será indicado explicitamente.

Este documento descreve apenas o primeiro recurso do addon: **Genetic Queen Filter**.

---

# 1. Objetivo do addon e escopo do MVP

O addon adicionará ao Frackin' Universe uma máquina chamada **Genetic Queen Filter**.

A máquina automatiza a seleção genética de **Queens** e **Young Queens** de abelha do FU que já tiveram seu genome analisado/inspecionado.

A máquina **não compara duas abelhas diretamente**. Em vez disso, mantém internamente um **perfil genético de referência**. Cada candidata válida colocada no Input é comparada contra esse perfil usando seis traits e um operador configurável por trait.

O objetivo é permitir seleção genética progressiva e automatizável sem alterar o sistema original de breeding do FU.

## 1.1 O que o MVP faz

O Genetic Queen Filter:

1. recebe uma candidata em um slot de Input;
2. verifica se os dois slots de saída estão vazios;
3. valida se o item é uma Queen ou Young Queen analisada e geneticamente legível;
4. se não existir perfil, usa a primeira candidata válida como baseline;
5. se já existir perfil, verifica se a candidata pertence à mesma linhagem genética (`family + subtype`);
6. compara seis traits contra as referências armazenadas;
7. envia a candidata para `Approved` ou `Rejected`;
8. somente candidatas aprovadas podem atualizar referências;
9. permite configurar cada trait como `>=`, `<=` ou `—` (ignorar);
10. funciona autonomamente com GUI fechada;
11. funciona como container real de 3 slots para jogadores e ITDs;
12. não consome energia.

## 1.2 O que o MVP NÃO faz

Não implementar:

- comparação direta Queen A vs Queen B;
- score, pesos ou média entre traits;
- seleção da “melhor” Queen em um storage upstream;
- fila interna de candidatas;
- buffer oculto;
- consumo de energia;
- processamento de drones;
- alteração do sistema de breeding do FU;
- geração/reparo de genomes inválidos;
- suporte a `workTime`;
- suporte a `miteResistance`;
- lógica especial para `mutationChance`;
- serialização do perfil para o item quando a máquina for quebrada;
- locks multiplayer próprios;
- semântica especial de Input/Output para ITDs;
- ScriptPane com slots proxy.

---

# 2. Runtime e dependências

## 2.1 OpenStarbound

**OpenStarbound é o runtime técnico de referência do projeto.**

Motivo principal: queremos uma **máquina que continue sendo um container verdadeiro**, com slots reais do engine, mas que também possua uma interface customizada/scriptável para perfil genético, comparadores e Reset.

A compatibilidade com Starbound retail/vanilla é secundária e não deve limitar a implementação do MVP, a menos que seja revisitada explicitamente.

Apesar disso, a lógica server-side da máquina deve preferir APIs convencionais de container e object scripts quando não houver necessidade de um recurso exclusivo do OpenStarbound.

Princípio:

```text
SERVER / OBJECT LOGIC
→ usar APIs normais de object/container sempre que possível

CLIENT / CONTAINER UI
→ pode aproveitar o runtime de referência OpenStarbound
```

## 2.2 Frackin' Universe

O addon depende formalmente de **FrackinUniverse**.

O identificador do mod principal deve ser verificado no metadata atual do FU antes de finalizar `_metadata`; o repositório oficial atual é a fonte de verdade.

O addon consome estruturas do FU, mas o FU nunca deve depender ou conhecer o addon.

Direção de dependência:

```text
OpenStarbound runtime
        ↓
Frackin Universe
        ↓
FUBeesGenetics addon
        ↓
Genetic Queen Filter
```

## 2.3 Sem dependência direta de metaGUI

A arquitetura oficial do MVP é:

```text
true container + scriptable container interface
```

Não usar metaGUI ou StardustLib diretamente a menos que uma limitação real da interface de container do runtime obrigue uma revisão arquitetural.

Essa revisão não deve ser feita silenciosamente: se for necessária, deve ser tratada como mudança de design.

---

# 3. Fontes técnicas de referência

A implementação deve consultar os arquivos atuais do FU, sem copiar lógica desnecessariamente.

## 3.1 FU — genética de abelhas

### `/bees/genomeLibrary.lua`

Funções/estruturas relevantes:

- `genelib.stringLibrary = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"`
- `genelib.statOrder`
- `genelib.statFromGenomeToDecimal(genome, stat)`
- `genelib.statFromGenomeToValue(genome, stat)`
- `genelib.statToDecimal(val)`
- `genelib.statToValue(val, stat)`

`statOrder` atual:

```text
subtype
baseProduction
droneToughness
droneBreedRate
queenBreedRate
queenLifespan
mutationChance
miteResistance
workTime
```

Com 9 genes × 2 caracteres, o genome atual esperado possui **18 caracteres**.

### `/bees/beeData.config`

Fonte de:

- families;
- subtypes por family;
- nome de cada subtype;
- defaults genéticos;
- `beeUpdateInterval` atual (= 1 s no momento da auditoria).

### `/bees/beeBuilder.lua`

Importante para entender:

- `genomeInspected` é o sinal de genome revelado/analisado;
- `wild` é informação distinta de `genomeInspected`;
- `parameters.lifespan` é vida restante da Queen, diferente do gene `queenLifespan`;
- tooltip oficial usa `genelib.returnFullGenomeStats()`;
- o subtype é resolvido dentro da family.

### `/bees/apiary.lua`

Importante para confirmar:

- uso de tags `bee`, `queen`, `youngQueen`, `drone`;
- Young Queen é uma entidade/item genético válido;
- o apiário pode converter Young Queen em Queen;
- o apiário pode gerar genome ausente em alguns fluxos — **nosso filtro não deve repetir isso**;
- family é extraída do nome interno da abelha;
- `transferUtil.lua` é usado pelo sistema de abelhas.

## 3.2 FU — automação

### `/scripts/kheAA/transferUtil.lua`

Função principal de interesse:

```text
transferUtil.loadSelfContainer()
```

Ela registra o próprio objeto/container no sistema de transferência.

O próprio arquivo possui exemplo/comentário de atualização aproximadamente uma vez por segundo.

### `/objects/power/isn_atmoscondenser/isn_resource_generator.lua`

Referência de máquina autônoma que:

- possui update periódico;
- usa `transferUtil.lua`;
- limita operações relacionadas a ITD em vez de executá-las desnecessariamente todo frame.

## 3.3 FU — Bee Station

### `/objects/bees/beestation/beestation.object`

Atualmente define:

```text
objectName = beestation
interactAction = OpenCraftingInterface
ui config = /interface/windowconfig/beestation.config
filter = beestation
```

Também contém um bloco antigo/comentado de `learnBlueprintsOnPickup`, útil como referência histórica do próprio FU.

### `/interface/windowconfig/beestation.config`

Pontos relevantes:

- `requiresBlueprint = true`;
- aba com filtro `functional`;
- aba com filtro `frame`.

### `/recipes/bees/normalapiary.recipe`

Referência para groups de equipamentos de abelha:

```text
mod
beestation
tools
all
functional
```

O Genetic Queen Filter deve seguir a aba `functional`.

## 3.4 FU — container real

### `/objects/power/centrifuge/centrifuge.object`

Referência importante de máquina FU:

- `objectType = container`;
- `slotCount` real;
- `uiConfig` de container;
- object script separado.

### `/interface/bees/industrialcentrifuge/industrialcentrifuge2.config`

Referência de interface que usa `itemgrid` + `slotOffset` para exibir grupos distintos do mesmo container real.

### `/objects/bees/centrifuge.lua`

Referência de object script operando diretamente em container real através de APIs `world.container*`.

## 3.5 OpenStarbound — comunicação entre UI e objeto

A interface customizada deve tratar o object script como autoridade e comunicar-se por mensagens assíncronas quando estiver cruzando contexto cliente/servidor.

Preferir:

```text
world.sendEntityMessage(...)
```

Evitar depender de `world.callScriptedEntity()` para GUI → objeto, pois chamadas síncronas exigem entidade local/master compatível e são menos apropriadas para multiplayer.

### Nota de implementação

A **sintaxe exata** usada pela versão de OpenStarbound instalada para anexar o script à interface de container deve ser confirmada no runtime/repositório no início da etapa de GUI.

Isso não muda a decisão arquitetural:

> deve ser um container verdadeiro com interface de container scriptável; não migrar para slots proxy apenas porque a sintaxe do config precisa ser localizada.

---

# 4. Namespace e IDs

## 4.1 Identificador do addon

Usar:

```text
FUBeesGenetics
```

como nome técnico principal do addon em metadata, salvo incompatibilidade comprovada com o sistema de metadata.

## 4.2 Namespace de assets

Usar prefixo:

```text
fubeesgenetics_
```

Objetivo: evitar colisões com FU e outros addons.

## 4.3 ID do primeiro objeto

```text
fubeesgenetics_geneticqueenfilter
```

Esse ID deve ser reutilizado de forma consistente em:

- `objectName`;
- output da recipe;
- unlock blueprint;
- referências internas pertinentes.

## 4.4 Namespace de mensagens

Usar:

```text
fubeesgenetics.gqf.getState
fubeesgenetics.gqf.setMode
fubeesgenetics.gqf.resetProfile
```

`gqf` = Genetic Queen Filter.

## 4.5 Traits — nomes internos

Manter os nomes originais do FU:

```text
baseProduction
droneToughness
droneBreedRate
queenBreedRate
queenLifespan
mutationChance
```

Não criar aliases internos desnecessários.

## 4.6 Modos internos

Persistir como:

```text
gte
lte
ignore
```

Nunca armazenar os símbolos visuais `>=`, `<=`, `—` como valor de estado.

## 4.7 Status operacionais

```text
WAITING_INPUT
WAITING_OUTPUTS
PROCESSING
```

Não é necessário `IDLE`; `WAITING_INPUT` já cobre esse estado.

## 4.8 Resultados

```text
NONE
PROFILE_INITIALIZED
APPROVED
REJECTED_INVALID_ITEM
REJECTED_NOT_INSPECTED
REJECTED_INVALID_GENOME
REJECTED_INCOMPATIBLE_SUBSPECIES
REJECTED_GENETIC_CRITERIA
```

---

# 5. Estrutura de arquivos planejada

Estrutura recomendada:

```text
FUBeesGenetics/
│
├── _metadata
│
├── objects/
│   └── bees/
│       ├── fubeesgenetics/
│       │   └── geneticqueenfilter/
│       │       ├── geneticqueenfilter.object
│       │       ├── geneticqueenfilter.png
│       │       ├── geneticqueenfiltericon.png
│       │       └── fubeesgenetics_geneticqueenfilter.lua
│       │
│       └── beestation/
│           └── beestation.object.patch
│
├── scripts/
│   └── fubeesgenetics/
│       └── fubeesgenetics_beegeneticadapter.lua
│
├── interface/
│   └── fubeesgenetics/
│       └── geneticqueenfilter/
│           ├── fubeesgenetics_geneticqueenfilter.config
│           ├── fubeesgenetics_geneticqueenfilterinterface.lua
│           └── <assets visuais da GUI>
│
└── recipes/
    └── fubeesgenetics/
        └── fubeesgenetics_geneticqueenfilter.recipe
```

### Importante

O patch do Bee Station precisa estar no **mesmo caminho virtual do asset original** que ele patcha.

Os demais arquivos devem ficar no namespace próprio do addon.

---

# 6. Objeto físico

## 6.1 Nome público

```text
Genetic Queen Filter
```

## 6.2 Descrição curta

Usar exatamente:

> **Filters inspected Queens and Young Queens using configurable genetic criteria.**

## 6.3 Tamanho

```text
5 tiles de largura × 4 tiles de altura
```

Na escala padrão de 8 px/tile, o footprint visual básico corresponde a aproximadamente 40 × 32 pixels, podendo haver margens transparentes conforme necessário.

## 6.4 Placement

- equipamento apoiado no chão;
- uma única orientação;
- sem flip esquerda/direita;
- sem versões alternativas.

Motivo: o layout visual associa permanentemente esquerda/centro/direita a Approved/Input/Rejected.

## 6.5 Sprite

O sprite deve ser **100% estático**.

Não implementar:

- frames de idle;
- frames de processamento;
- LEDs que mudam;
- luz piscante;
- animação de scanner;
- partículas;
- alteração visual quando output ocupa slot.

Motivo: reduzir custo visual/performance e manter feedback dinâmico somente na GUI.

## 6.6 Iluminação

Não adicionar iluminação dinâmica real (`lightColor` etc.) no MVP.

Pode haver pixels ciano desenhados como elementos tecnológicos, mas sem fonte de luz do engine.

## 6.7 Linguagem visual

Visual pretendido:

- estrutura metálica escura;
- acentos azul/ciano;
- detalhe permanente verde à esquerda;
- detalhe permanente vermelho à direita;
- câmara/scanner central com aparência de reinforced glass;
- motivos discretos de DNA/hexágonos/abelhas;
- nenhuma Queen permanentemente desenhada dentro da câmara;
- nenhum texto legível no sprite.

Motivo para não desenhar uma Queen: a máquina pareceria ocupada mesmo vazia.

## 6.8 Convenção visual dos lados

Visualmente:

```text
LEFT       CENTER      RIGHT
Approved   Input       Rejected
verde      ciano       vermelho
```

Isso é apenas linguagem visual da máquina. Não cria restrições de I/O para ITD.

## 6.9 Ícone de inventário

Ícone próprio, derivado da silhueta da máquina:

- corpo escuro;
- elemento ciano central;
- pequenos detalhes verde/vermelho;
- sem texto;
- sem necessidade de mostrar os três slots.

---

# 7. Inspeção por raça

Seguir o padrão do Starbound em que diferentes raças possuem descrições específicas ao inspecionar o mesmo objeto.

Textos definidos:

### Human

> A machine that compares analyzed queen bees and sorts them by their genetic traits. Pretty clever.

### Apex

> A selective genetics apparatus. It compares each specimen against precisely configured genetic parameters.

### Avian

> A device for guiding generations of queens toward chosen traits. An unusual kind of artificial selection.

### Floran

> Machine judgesss queen beesss. Strong genes go one way, bad genes go other way.

Observação: “strong/bad” é propositalmente uma interpretação simplificada do Floran; não descreve a semântica técnica precisa de `>=`/`<=`.

### Glitch

> Analytical. This machine compares the genetic traits of queen bees against a stored reference profile.

### Hylotl

> A precise instrument for selectively refining the traits of bee queens across generations.

### Novakid

> This contraption sorts queen bees by their genes. Saves a whole lotta pickin' through 'em by hand.

### Generic/fallback

> Stores a genetic reference and sorts inspected bee queens according to configurable traits.

---

# 8. Container e contrato dos slots

## 8.1 Deve ser container real

O objeto deve ser declarado como um **container verdadeiro**, seguindo o modelo de máquinas como Centrifuge do FU.

Requisitos:

```text
objectType = container
slotCount = 3
```

Os slots precisam existir como slots reais do engine.

## 8.2 Slots lógicos

Documentação/user-facing:

```text
Slot 1 — Input
Slot 2 — Approved
Slot 3 — Rejected
```

## 8.3 Offsets internos

APIs de container usam indexação zero-based.

Logo:

```text
offset 0 — Input
offset 1 — Approved
offset 2 — Rejected
```

Manter essa tradução explícita para evitar bugs de off-by-one.

## 8.4 Ordem visual na GUI

A GUI deve mostrar:

```text
Approved | Input | Rejected
```

Isso não exige mudar offsets. Usar `itemgrid`/`slotOffset` adequadamente.

Conceitualmente:

```text
grid esquerda → offset 1
grid centro   → offset 0
grid direita  → offset 2
```

## 8.5 ITD

Para ITDs, a máquina deve ser vista apenas como:

> **storage/container comum de três slots**.

A máquina não expõe nem impõe semântica de direção para automação.

Toda escolha de qual slot será usado para inserir ou retirar é responsabilidade da configuração do ITD.

Isso significa que um ITD mal configurado pode:

- inserir em Approved;
- inserir em Rejected;
- retirar de Input.

Isso não é bug da máquina. É configuração do jogador.

## 8.6 Nenhum slot protegido

A GUI também não deve bloquear manualmente a escrita em Approved/Rejected.

Se o jogador colocar algo em Approved, o slot está ocupado e a máquina passa a esperar.

---

# 9. Regra fundamental de processamento

A máquina **somente inicia um ciclo de processamento se ambos os outputs estiverem vazios**.

Pré-condição:

```text
Input ocupado
AND
Approved vazio
AND
Rejected vazio
```

Se Approved **ou** Rejected estiver ocupado:

```text
→ WAITING_OUTPUTS
→ NÃO validar Input
→ NÃO ler genome
→ NÃO comparar
→ NÃO atualizar memória
→ NÃO mover item
```

Isso é obrigatório mesmo se o item do output pudesse formar stack.

A regra é **slot vazio vs slot ocupado**, não “há espaço no slot”.

## 9.1 Por que essa regra existe

Ela cria backpressure natural e elimina a necessidade de:

- fila interna;
- resultado pendente;
- buffer oculto;
- gerenciamento complexo de vários outputs.

O item aprovado/rejeitado deve ser removido antes de um novo ciclo.

---

# 10. Quantidade processada por ciclo

Processar **uma unidade** do stack de Input por ciclo.

Se o Input contiver uma stack de várias Queens/Young Queens:

1. analisar o descriptor atual;
2. decidir o resultado;
3. retirar exatamente 1 unidade;
4. colocar a unidade no output;
5. como o output ficou ocupado, processamento para até ele ser esvaziado.

Não processar a stack inteira de uma vez.

---

# 11. Bee Genetic Adapter — objetivo

Criar um script dedicado, por exemplo:

```text
/scripts/fubeesgenetics/fubeesgenetics_beegeneticadapter.lua
```

Responsabilidade única:

> converter um `ItemDescriptor` do FU em dados genéticos normalizados ou em um motivo de falha.

Ele deve ser **somente leitura**.

## 11.1 O Adapter NÃO conhece

O Adapter não deve saber nada sobre:

- slots;
- perfil da máquina;
- Approved/Rejected;
- comparadores;
- Reset;
- `storage` da máquina;
- ITD;
- GUI.

## 11.2 O Adapter NÃO deve mutar itens

Proibido:

- gerar genome;
- reparar genome;
- definir `genomeInspected`;
- transformar Young Queen em Queen;
- modificar lifespan;
- alterar parâmetros.

Isso difere deliberadamente do Apiary do FU, que possui fluxos próprios de manutenção/produção e pode gerar genome ausente.

Nosso filtro é um **classificador**, não um reparador.

---

# 12. Bee Genetic Adapter — entrada

Entrada:

```text
ItemDescriptor
```

Conceitualmente:

```text
item.name
item.count
item.parameters
```

Parâmetros potencialmente presentes:

```text
genome
genomeInspected
wild
lifespan
...
```

Para genética do filtro:

- `count`: não altera genética;
- `wild`: ignorado;
- `lifespan`: ignorado;
- `genome`: essencial;
- `genomeInspected`: essencial.

---

# 13. Elegibilidade de item

Somente são candidatas geneticamente elegíveis:

- Queen analisada;
- Young Queen analisada.

Verificar por tags do FU, não por substring textual do nome.

Critério conceitual:

```text
root.itemHasTag(item.name, "bee")
AND
(
  root.itemHasTag(item.name, "queen")
  OR
  root.itemHasTag(item.name, "youngQueen")
)
AND
genomeInspected
AND
genome válido
```

## 13.1 Itens que devem ser rejeitados

Diretamente para Rejected:

- drones;
- outras castas;
- itens não-bee;
- Queen não analisada;
- Young Queen não analisada;
- genome ausente;
- genome malformado;
- family desconhecida;
- subtype inválido.

Nenhum desses casos pode alterar referências.

---

# 14. `wild` e `genomeInspected`

**Não usar `wild` para decidir se a abelha está analisada.**

No FU, `wild` e `genomeInspected` são parâmetros separados.

Logo:

```text
wild = true
genomeInspected = true
→ elegível
```

E:

```text
wild = false
genomeInspected ausente/false
→ não elegível
```

Motivo: `beeBuilder.lua` usa `genomeInspected` para revelar os stats e trata `wild` separadamente.

---

# 15. Validação estrutural do genome

Antes de usar funções que assumem caracteres válidos, validar o genome.

Formato atual esperado:

- tipo string;
- exatamente 18 caracteres;
- somente caracteres de `0-9A-Z`.

A biblioteca atual do FU usa:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Cada 2 caracteres codificam um valor de 0 a 1295.

## 15.1 Por que validar antes

`statToDecimal()` usa busca direta no alfabeto. Caracteres inválidos podem resultar em erro de Lua em vez de retorno controlado.

O Adapter deve impedir que um ItemDescriptor corrompido derrube o object script.

---

# 16. Ordem dos genes

A ordem atual do FU é:

```text
1  subtype
2  baseProduction
3  droneToughness
4  droneBreedRate
5  queenBreedRate
6  queenLifespan
7  mutationChance
8  miteResistance
9  workTime
```

Com dois caracteres por gene.

A implementação deve preferir as funções de `genomeLibrary.lua` em vez de espalhar offsets hardcoded sempre que possível.

---

# 17. Identidade de linhagem: `family + subtype`

## 17.1 Family

O FU usa nomes de item de abelha onde a family pode ser extraída do trecho entre underscores, padrão observado no próprio código de bees.

Exemplo conceitual:

```text
bee_<family>_queen
bee_<family>_youngQueen
```

A implementação deve replicar **a convenção atual do FU**, preferencialmente encapsulada em uma função do Adapter.

Depois de extrair family:

```text
beeData.stats[family]
```

precisa existir.

Se não existir:

```text
UNKNOWN_FAMILY → tratar como INVALID_GENOME para a máquina
```

## 17.2 Subtype

`subtype` é o primeiro gene e não é globalmente único.

Ele funciona como índice dentro da lista da family.

Portanto:

```text
identity = family + subtype
```

Nunca usar subtype isoladamente.

## 17.3 Validade do subtype

Apesar de o gene poder decodificar `00` como 0, subtype válido do FU é indexado de 1 até o número de subtypes da family.

Validação:

```text
subtype >= 1
AND
subtype <= #beeData.stats[family]
```

Caso contrário:

```text
INVALID_SUBTYPE
```

mapeado para `REJECTED_INVALID_GENOME`.

## 17.4 `subspeciesName`

Resolver via:

```text
beeData.stats[family][subtype].name
```

Esse nome serve somente para GUI.

Identidade persistente real continua sendo:

```text
family + subtype
```

Não usar `subspeciesName` para comparação.

## 17.5 Queen vs Young Queen

Não faz parte da identidade.

Uma Young Queen pode estabelecer baseline e uma Queen adulta da mesma `family + subtype` deve ser aceita para comparação, e vice-versa.

---

# 18. Traits suportados

Exatamente seis:

```text
baseProduction
droneToughness
droneBreedRate
queenBreedRate
queenLifespan
mutationChance
```

Não adicionar outros traits no MVP.

## 18.1 Observação sobre traits com “drone” no nome

`droneToughness` e `droneBreedRate` são lidos do **genome da Queen/Young Queen**.

Isso não significa que drones sejam candidatos.

Drones continuam proibidos como input genético.

---

# 19. Traits excluídos

## 19.1 `miteResistance`

Excluir totalmente do MVP.

Motivo: embora o gene ainda exista, o sistema de mites do estado atual do FU não justifica o trait para esta máquina.

Não armazenar, comparar nem mostrar.

## 19.2 `workTime`

Completamente fora do projeto.

Não armazenar, comparar, mostrar ou usar em identidade.

---

# 20. Raw vs Display

Para cada um dos seis traits, o Adapter deve fornecer:

```text
raw
display
```

## 20.1 Raw

Obter com:

```text
genelib.statFromGenomeToDecimal(genome, stat)
```

Usar para:

- comparação;
- persistência da referência;
- atualização de referência.

## 20.2 Display

Obter com:

```text
genelib.statFromGenomeToValue(genome, stat)
```

Usar somente para GUI.

## 20.3 Motivo

Evitar arredondamento e inconsistências de apresentação.

`mutationChance`, por exemplo, possui conversão própria de raw para valor exibido.

A GUI nunca deve determinar aprovação com base no valor formatado.

---

# 21. Contrato de retorno do Adapter

## 21.1 Sucesso

Conceitualmente:

```text
{
  eligible = true,
  identity = {
    family = <string>,
    subtype = <integer>,
    subspeciesName = <string>
  },
  genetics = {
    baseProduction = { raw = ..., display = ... },
    droneToughness = { raw = ..., display = ... },
    droneBreedRate = { raw = ..., display = ... },
    queenBreedRate = { raw = ..., display = ... },
    queenLifespan = { raw = ..., display = ... },
    mutationChance = { raw = ..., display = ... }
  }
}
```

Não é necessário devolver o genome inteiro ao Machine Object se ele não for usado fora do Adapter.

## 21.2 Falhas internas do Adapter

Recomendado:

```text
INVALID_DESCRIPTOR
NOT_BEE
NOT_QUEEN
NOT_INSPECTED
INVALID_GENOME
UNKNOWN_FAMILY
INVALID_SUBTYPE
```

Mapeamento para resultados da máquina:

```text
INVALID_DESCRIPTOR → REJECTED_INVALID_ITEM
NOT_BEE            → REJECTED_INVALID_ITEM
NOT_QUEEN          → REJECTED_INVALID_ITEM
NOT_INSPECTED      → REJECTED_NOT_INSPECTED
INVALID_GENOME     → REJECTED_INVALID_GENOME
UNKNOWN_FAMILY     → REJECTED_INVALID_GENOME
INVALID_SUBTYPE    → REJECTED_INVALID_GENOME
```

Não retornar `INCOMPATIBLE_SUBSPECIES` do Adapter, pois incompatibilidade depende do perfil armazenado na máquina.

---

# 22. Perfil persistente da máquina

Estado persistente mínimo:

```text
storage.profile
storage.references
storage.modes
storage.operationalStatus
storage.lastResult
```

## 22.1 `storage.profile`

```text
family
subtype
subspeciesName
```

Ausência de perfil deve ser `nil`/estrutura ausente, nunca valores sentinela como `0`.

## 22.2 `storage.references`

Uma entrada raw para cada um dos seis traits.

Exemplo conceitual:

```text
references = {
  baseProduction = 500,
  droneToughness = 300,
  droneBreedRate = 700,
  queenBreedRate = 450,
  queenLifespan = 900,
  mutationChance = 250
}
```

## 22.3 `storage.modes`

Um dos valores:

```text
gte
lte
ignore
```

por trait.

Default, quando não existir estado anterior:

```text
gte
```

para todos os seis traits.

---

# 23. Comparadores

Cada trait tem exatamente três modos.

## 23.1 `gte` / `>=`

Passa se:

```text
candidate.raw >= reference.raw
```

Igualdade passa.

## 23.2 `lte` / `<=`

Passa se:

```text
candidate.raw <= reference.raw
```

Igualdade passa.

## 23.3 `ignore` / `—`

O trait:

- não participa da aprovação;
- nunca causa rejeição;
- **não atualiza sua referência** quando a candidata é aprovada.

A referência fica congelada.

Se o usuário reativar o trait depois, o valor antigo volta a ser usado.

## 23.4 Mutation Chance

`mutationChance` segue exatamente as mesmas regras.

Não implementar inversão automática, default diferente ou lógica “menor é melhor”.

O jogador escolhe explicitamente `>=`, `<=` ou `—`.

---

# 24. Baseline — primeira candidata válida

Se não existir `storage.profile`:

1. validar a candidata normalmente;
2. não comparar contra nada;
3. definir `family`;
4. definir `subtype`;
5. definir `subspeciesName`;
6. copiar os **seis raw values** para `storage.references`;
7. enviar a candidata para Approved;
8. `lastResult = PROFILE_INITIALIZED`.

## 24.1 Todos os seis traits recebem referência

Mesmo que um ou mais modos estejam em `ignore`, a primeira baseline deve armazenar os seis raw values.

Isso garante que um trait ignorado possa ser reativado posteriormente com uma referência válida.

## 24.2 A primeira candidata é sempre Approved

Desde que seja uma Queen/Young Queen analisada, com genome válido.

Não há comparação durante baseline.

---

# 25. Compatibilidade de linhagem

Quando já existe perfil:

```text
candidate.family == profile.family
AND
candidate.subtype == profile.subtype
```

Se qualquer um falhar:

```text
REJECTED_INCOMPATIBLE_SUBSPECIES
```

Não executar comparação dos seis traits.

Não alterar referências.

---

# 26. Regra de aprovação

Para uma candidata compatível:

- avaliar somente traits não ignorados;
- **todos** os traits ativos precisam passar.

Lógica AND estrita:

```text
pass = trait1 AND trait2 AND ... AND traitN
```

Não implementar:

- score;
- pesos;
- tolerância;
- média;
- compensação de um trait por outro.

Se um único trait ativo falhar:

```text
REJECTED_GENETIC_CRITERIA
```

---

# 27. Regra absoluta das rejeitadas

> **Uma candidata rejeitada NUNCA modifica nenhuma referência genética.**

Essa é uma invariante crítica.

Exemplo:

```text
Reference:
Production >= 100
Toughness  >= 100

Candidate:
Production = 500
Toughness  = 99
```

Resultado:

```text
Rejected
```

Referências permanecem:

```text
Production = 100
Toughness  = 100
```

Nunca aproveitar o `500` da candidata rejeitada.

## 27.1 Por que

O perfil representa apenas a progressão obtida por indivíduos que passaram **integralmente** pela seleção ativa.

Aproveitar ganhos parciais de uma rejeitada mudaria completamente a semântica de seleção e criaria um algoritmo de otimização por trait que não foi escolhido para o projeto.

---

# 28. Atualização após aprovação

Somente depois de uma candidata ser efetivamente enviada para Approved:

Para cada trait:

```text
mode == gte    → reference = candidate.raw
mode == lte    → reference = candidate.raw
mode == ignore → reference permanece inalterada
```

## 28.1 Perfil composto

As referências podem formar um conjunto que nunca existiu em uma única Queen.

Exemplo:

```text
Ref:
Production = 100
Lifespan = 500

Modes:
Production = gte
Lifespan = ignore

Approved candidate:
Production = 150
Lifespan = 300
```

Novo perfil:

```text
Production = 150
Lifespan = 500
```

Isso é intencional.

Não armazenar “última Queen aprovada” como fonte absoluta do perfil.

---

# 29. Caso: todos os traits ignorados

Configuração válida:

```text
ignore
ignore
ignore
ignore
ignore
ignore
```

Com perfil existente, qualquer Queen/Young Queen válida da mesma `family + subtype` é Approved.

Como todos estão ignorados:

```text
nenhuma referência muda
```

---

# 30. Reset Profile

Botão:

```text
Reset Profile
```

## 30.1 Comportamento

Um clique executa imediatamente.

**Sem diálogo de confirmação.**

Apaga:

```text
storage.profile
storage.references
```

Preserva:

```text
storage.modes
conteúdo dos 3 slots
```

Recomendado:

```text
storage.lastResult = NONE
```

Status operacional deve ser recalculado normalmente conforme os slots.

## 30.2 Reset pode ocorrer com slots ocupados

Não bloquear Reset devido a Input/Approved/Rejected.

Se uma candidata válida já estiver no Input, ela poderá virar nova baseline quando ambos outputs estiverem vazios e chegar o próximo ciclo.

---

# 31. Machine Object — autoridade

O object script é a autoridade de:

- perfil;
- referências;
- modos;
- processamento;
- movimentação automática de itens;
- resultados/status;
- persistência;
- mensagens da GUI;
- registro no sistema ITD.

A GUI não implementa lógica genética.

---

# 32. Machine Object — polling

Baseline técnico:

```text
1 processing check por segundo
```

Motivos:

- coerente com `beeUpdateInterval = 1` no FU atual;
- responsivo o suficiente para automação;
- evita executar genética todo frame;
- escala melhor com múltiplas máquinas.

## 32.1 Ajustável

O valor de 1 s pode ser refinado durante testes se houver motivo técnico concreto.

O requisito real é:

> processamento periódico, autônomo e independente da GUI.

Não é necessário simular ciclos enquanto o mundo/chunk não está carregado.

---

# 33. Ordem exata do ciclo de processamento

## 33.1 Passo 1 — outputs

Primeiro verificar:

```text
Approved vazio?
Rejected vazio?
```

Se qualquer um ocupado:

```text
operationalStatus = WAITING_OUTPUTS
RETURN
```

Não olhar genética do Input.

## 33.2 Passo 2 — Input

Se Input vazio:

```text
operationalStatus = WAITING_INPUT
RETURN
```

## 33.3 Passo 3 — snapshot

Ler o descriptor atual do Input sem retirar.

Enviar ao Adapter.

## 33.4 Passo 4 — decidir resultado

Determinar logicamente:

- Rejected invalid item;
- Rejected not inspected;
- Rejected invalid genome;
- Baseline;
- Rejected incompatible subtype/family;
- Rejected genetic criteria;
- Approved.

Também calcular, em memória temporária, qual seria o próximo perfil/referências **sem ainda gravar**.

## 33.5 Passo 5 — revalidar antes de mover

Imediatamente antes de retirar a unidade, reler/revalidar o Input e confirmar que ainda corresponde à candidata analisada.

Motivo: outro jogador ou ITD pode ter alterado o container entre operações.

Se mudou:

```text
abort cycle
```

Sem alteração genética.

Também confirmar outputs ainda vazios.

## 33.6 Passo 6 — retirar exatamente uma unidade

Usar API por slot adequada, por exemplo equivalente a:

```text
containerTakeNumItemsAt(entity.id(), INPUT_OFFSET, 1)
```

Se nada for retirado:

```text
abort
```

## 33.7 Passo 7 — inserir no output escolhido

Destination:

```text
APPROVED_OFFSET ou REJECTED_OFFSET
```

Como o output foi verificado vazio, a unidade normalmente deve caber.

## 33.8 Passo 8 — confirmar inserção

Somente considerar o movimento concluído se a inserção foi confirmada sem leftover inesperado.

## 33.9 Passo 9 — commit

Somente agora:

- inicializar profile/references se baseline;
- ou atualizar referências ativas se Approved;
- ou apenas atualizar `lastResult` se Rejected.

---

# 34. Atomicidade — invariante crítica

> **Uma referência só pode mudar se a Queen correspondente já tiver sido inserida com sucesso no slot Approved.**

Nunca:

```text
atualizar memória
↓
tentar mover item
```

Sempre:

```text
mover item com sucesso
↓
commit da memória
```

## 34.1 Falha de inserção

Se a unidade foi retirada do Input mas a inserção no output falhar:

1. não modificar profile/references;
2. tentar rollback da unidade para o Input;
3. abortar o ciclo.

Não apagar item silenciosamente.

## 34.2 Não implementar journal complexo no MVP

Não criar inicialmente:

- transaction log persistente;
- pending-result queue;
- recovery journal.

A atomicidade exigida é lógica durante operação normal.

Se testes reais demonstrarem problema de interrupção entre calls síncronas, reavaliar separadamente.

---

# 35. Persistência

Enquanto o objeto estiver colocado, usar `storage` normal do object script.

Persistir:

- profile;
- references;
- modes;
- lastResult;
- status quando útil.

Não persistir desnecessariamente:

- genome inteiro;
- última Queen;
- fila;
- item pendente;
- `workTime`;
- `miteResistance`.

## 35.1 Ao quebrar a máquina

É aceitável perder:

- perfil;
- referências;
- modos.

Não implementar serialização especial desses dados no item dropado.

Itens físicos do container devem seguir o comportamento normal do container/engine ao destruir a máquina.

---

# 36. Integração ITD

Carregar:

```text
/scripts/kheAA/transferUtil.lua
```

Registrar o container via:

```text
transferUtil.loadSelfContainer()
```

em intervalo adequado, usando padrão semelhante a máquinas do FU.

Baseline:

```text
~1 vez por segundo
```

Pode compartilhar timer com processamento se a implementação ficar clara, ou usar acumuladores separados.

## 36.1 Não implementar semântica de slot para ITD

O addon não deve informar ao ITD “este é Input” ou “este é Output”.

Essa configuração pertence ao ITD.

---

# 37. Interface — arquitetura obrigatória

**Não usar ScriptPane com ItemSlotWidgets simulando os três slots.**

Usar:

> **container real de 3 slots + interface de container scriptável.**

Razões:

- slots continuam nativos;
- stacking continua nativo;
- mouse/cursor continua nativo;
- clique direito continua nativo;
- multiplayer de container continua nativo;
- ITD usa os mesmos slots;
- não duplicamos lógica de inventário no cliente.

A Centrifuge do FU é referência do princípio `objectType=container` + `uiConfig` + `itemgrid` real.

---

# 38. Interface — layout funcional

Estrutura:

```text
Genetic Queen Filter

GENETIC PROFILE
Family: <value or —>
Subspecies: <value or —>       [Reset Profile]

STAT                  REFERENCE     >=   <=   —
Production             <value>       ○    ○    ○
Drone Toughness        <value>       ○    ○    ○
Drone Breed Rate       <value>       ○    ○    ○
Queen Breed Rate       <value>       ○    ○    ○
Queen Lifespan         <value>       ○    ○    ○
Mutation Chance        <value>       ○    ○    ○

[Approved]              [Input]              [Rejected]

Status: ...
Last result: ...
```

## 38.1 Valores antes da baseline

Mostrar:

```text
Family: —
Subspecies: —
Reference: —
```

Nunca mostrar `0` para “não inicializado”, pois zero é valor genético válido.

## 38.2 Subtype numérico

Não mostrar o número do subtype para o jogador.

Exibir:

```text
Family
SubspeciesName
```

O número permanece interno.

---

# 39. Interface — valores de referência

Mostrar os valores **display** convertidos pelo FU.

Não mostrar raw ao usuário.

Exemplo:

```text
mutationChance.raw = 537
```

pode virar uma porcentagem na UI conforme `statFromGenomeToValue()`.

A interface jamais recalcula comparação a partir do valor exibido.

---

# 40. Interface — controles dos comparadores

Cada trait possui um grupo exclusivo:

```text
>=   <=   —
```

Somente uma opção ativa por trait.

Mapeamento:

```text
>= → gte
<= → lte
—  → ignore
```

A UI pode usar radio groups ou controles equivalentes do runtime.

Ao alterar:

```text
UI → fubeesgenetics.gqf.setMode → object script valida → storage.modes altera
```

A UI deve considerar o estado autoritativo somente após resposta/refresh do objeto.

---

# 41. Interface — Reset

Um botão:

```text
Reset Profile
```

Um clique:

```text
UI → fubeesgenetics.gqf.resetProfile
```

Sem confirmação.

Depois, redesenhar a partir do estado retornado/consultado.

---

# 42. Interface — mensagens

## 42.1 `fubeesgenetics.gqf.getState`

Retorna snapshot conceitual:

```text
initialized
profile:
  family
  subtype
  subspeciesName
traits:
  <trait>:
    referenceDisplay
    mode
operationalStatus
lastResult
```

Não é necessário retornar slots, porque o ContainerPane real já sincroniza o container nativamente.

## 42.2 `fubeesgenetics.gqf.setMode`

Argumentos:

```text
trait
mode
```

Object script valida:

- trait está na allowlist dos seis;
- mode está em `{gte,lte,ignore}`.

Não aceitar nomes arbitrários.

## 42.3 `fubeesgenetics.gqf.resetProfile`

Sem argumentos funcionais necessários.

Executa Reset definido anteriormente.

---

# 43. Interface — comunicação assíncrona

Preferir:

```text
world.sendEntityMessage(containerEntityId, messageName, ...)
```

pois a interface é cliente e o objeto pode ser masterizado pelo servidor.

Evitar criar uma dependência de chamadas síncronas cross-context.

## 43.1 Refresh

Recomendação:

- `getState` imediato ao abrir;
- refresh passivo aproximadamente a cada 0,5 s enquanto pane aberto;
- refresh imediato após `setMode`/`resetProfile` quando a resposta chegar.

## 43.2 Evitar spam de promises

Se houver um `getState` ainda pendente:

```text
não disparar outro
```

Manter no máximo uma consulta passiva pendente por interface.

---

# 44. Status e textos da GUI

## 44.1 `operationalStatus`

Mapeamento amigável:

```text
WAITING_INPUT   → Waiting for input
WAITING_OUTPUTS → Waiting for outputs to be cleared
PROCESSING      → Processing...
```

## 44.2 `lastResult`

```text
NONE                             → —
PROFILE_INITIALIZED              → Genetic profile initialized
APPROVED                         → Approved
REJECTED_INVALID_ITEM            → Rejected — invalid item
REJECTED_NOT_INSPECTED           → Rejected — genome not inspected
REJECTED_INVALID_GENOME          → Rejected — invalid genome
REJECTED_INCOMPATIBLE_SUBSPECIES → Rejected — incompatible subspecies
REJECTED_GENETIC_CRITERIA        → Rejected — genetic criteria
```

Não é necessário no MVP mostrar exatamente qual trait falhou.

---

# 45. Tooltips de comparadores

Recomendados:

### `>=`

> Accept if candidate value is equal to or greater than the reference.

### `<=`

> Accept if candidate value is equal to or lower than the reference.

### `—`

> Ignore this trait. Its stored reference will not change.

### Reference

> Genetic value currently used for comparison.

---

# 46. Cores da GUI

Semântica:

- Approved → verde;
- Input → ciano/neutro;
- Rejected → vermelho.

Não dar significado “bom/ruim” às cores dos comparadores.

`>=` e `<=` não devem receber verde/vermelho por padrão.

---

# 47. Multiplayer

Seguir a lógica normal de máquinas/container do FU/Starbound/OpenStarbound.

## 47.1 Múltiplas interfaces

Dois ou mais jogadores podem abrir a mesma máquina simultaneamente.

Não criar:

- ownership de GUI;
- lock exclusivo;
- reserva de slots;
- mutex de jogador;
- sessão separada por jogador.

Todos observam/manipulam:

```text
mesmo container
mesmo profile
mesmas references
mesmos modes
```

## 47.2 GUI é view, não autoridade

Cada jogador possui sua pane local, mas ela consulta o mesmo object script.

Se A muda um modo, B deve ver a mudança no próximo refresh.

## 47.3 Last-write-wins natural

Se dois comandos de configuração chegarem:

```text
A: Production = gte
B: Production = lte
```

O último comando aplicado pelo object script é o estado final.

Não implementar merge/conflict dialog.

## 47.4 Slots simultâneos

Jogadores e ITDs operam sobre o mesmo container real.

Não criar uma cópia de inventário por pane.

---

# 48. Crafting

## 48.1 Estação

Craftar em:

```text
Bee Station
```

## 48.2 Aba

Mesma aba dos maquinários/colmeias:

```text
functional
```

Seguir groups de referência do FU, por exemplo:

```text
mod
beestation
tools
all
functional
```

Confirmar contra recipes atuais antes de finalizar.

## 48.3 Receita

Definição de design:

```text
2 × AI Chip
5 × Laser Diode
5 × Reinforced Glass
10 × Titanium Bar
→ 1 × Genetic Queen Filter
```

### Importante — IDs internos

Os nomes acima são nomes de design/ingame. **Antes de escrever a recipe final, localizar nos assets atuais os item IDs exatos** de:

- AI Chip;
- Laser Diode;
- Reinforced Glass;
- Titanium Bar.

Não adivinhar IDs.

Registrar no commit/nota de implementação quais IDs foram confirmados.

## 48.4 Por que esses materiais

- AI Chips → processamento/decisão genética;
- Laser Diodes → scanner/leitura;
- Reinforced Glass → câmara óptica;
- Titanium Bars → estrutura física.

A máquina não usa energia, então balanceamento ocorre principalmente pela receita/progressão.

---

# 49. Desbloqueio da receita

Regra de design:

> A receita é aprendida quando o jogador obtém uma Bee Station no inventário.

Não criar:

- research node próprio;
- quest;
- unlock tecnológico separado.

## 49.1 Implementação planejada

Adicionar blueprint do Genetic Queen Filter ao comportamento de pickup da Bee Station por patch do asset do FU.

O Bee Station atual possui `requiresBlueprint = true`, então o aprendizado é necessário para a recipe aparecer.

## 49.2 Saves existentes

Se o addon for instalado em um save onde o jogador já tinha obtido Bee Station, o pickup antigo não será retroativamente reexecutado.

Aceitar esse edge case no MVP.

Solução documentada ao jogador/desenvolvedor:

> obter/recolher novamente uma Bee Station para disparar o blueprint.

Não adicionar player script global só para corrigir esse caso.

---

# 50. Energia

A máquina **não consome energia**.

Não implementar:

- power node;
- buffer;
- `fupower` apenas por conveniência;
- estado `NO_POWER`;
- consumo por ciclo.

Se balanceamento adicional for necessário, fazê-lo por recipe/progressão, não energia.

---

# 51. Performance

Decisões tomadas especificamente para reduzir custo:

- sprite estático;
- sem animação;
- sem luz dinâmica;
- processamento genético ~1 Hz;
- ITD registration ~1 Hz;
- GUI refresh somente enquanto aberta;
- sem scanning de storages externos;
- sem fila interna;
- sem simulação offline/unloaded própria.

Não otimizar prematuramente com caches complexos antes de medir.

---

# 52. Ordem de implementação recomendada

A implementação deve ser incremental.

## Fase 1 — estrutura mínima

Criar:

- `_metadata`;
- diretórios;
- placeholders mínimos.

Critério:

> addon carrega com FU + OpenStarbound sem erros de assets.

## Fase 2 — Bee Genetic Adapter

Implementar e validar isoladamente:

- tags;
- genomeInspected;
- genome validation;
- family;
- subtype;
- subspeciesName;
- seis traits raw/display.

Critério:

> uma Queen/Young Queen real do FU gera dados normalizados corretos.

## Fase 3 — container mínimo

Criar objeto verdadeiro:

```text
objectType = container
slotCount = 3
```

Ainda sem lógica genética completa.

Validar:

- jogadores usam os slots;
- stacks funcionam;
- multiplayer básico;
- ITD enxerga storage.

Critério:

> comporta-se como container comum de três slots.

## Fase 4 — motor genético

Implementar:

- polling;
- gating de outputs;
- baseline;
- compatibilidade;
- comparadores;
- aprovação/rejeição;
- commit seguro;
- persistência;
- Reset.

Critério:

> máquina funciona corretamente com GUI fechada.

## Fase 5 — ITD

Integrar `transferUtil` e validar inserção/extração em slots escolhidos pelo ITD.

Critério:

> máquina é storage de 3 slots normal para automação.

## Fase 6 — container interface scriptável

Somente depois do motor estar sólido:

- Family/Subspecies;
- referências;
- comparadores;
- Reset;
- status;
- lastResult;
- três slots reais em ordem visual Approved/Input/Rejected.

Critério:

> dois jogadores podem abrir simultaneamente e ver estado compartilhado.

## Fase 7 — crafting/unlock

- confirmar IDs dos materiais;
- recipe;
- groups;
- patch de blueprint na Bee Station.

Critério:

> obter Bee Station → aprender recipe → craftar a máquina.

## Fase 8 — arte e acabamento

- sprite final 5×4;
- icon;
- GUI final baseada no mockup aprovado;
- inspection texts;
- tooltips;
- polimento visual.

---

# 53. Matriz de testes obrigatória

## 53.1 Adapter

### A1 — Queen analisada

Esperado:

- eligible = true;
- family correta;
- subtype correto;
- seis traits raw/display.

### A2 — Young Queen analisada

Mesmo resultado estrutural de Queen válida.

### A3 — Drone

Esperado:

```text
NOT_QUEEN / REJECTED_INVALID_ITEM
```

### A4 — item não-bee

Rejected invalid item.

### A5 — Queen não analisada

Rejected not inspected.

### A6 — wild + inspected

Deve ser elegível.

### A7 — non-wild + not inspected

Não elegível.

### A8 — genome ausente

Rejected invalid genome.

### A9 — genome com tamanho errado

Rejected invalid genome.

### A10 — genome com caractere fora de `0-9A-Z`

Rejected invalid genome sem crash.

### A11 — subtype 0

Rejected invalid genome.

### A12 — subtype maior que a lista da family

Rejected invalid genome.

---

# 54. Testes de baseline

### B1 — primeira Queen válida

- outputs vazios;
- perfil vazio;
- Queen válida no Input.

Esperado:

- 1 unidade vai para Approved;
- family/subtype armazenados;
- todos os seis raw values armazenados;
- lastResult = PROFILE_INITIALIZED.

### B2 — primeira Young Queen válida

Mesmo comportamento.

### B3 — traits em ignore durante baseline

Mesmo assim armazenar todas as seis referências.

---

# 55. Testes dos comparadores

### C1 — igualdade em `gte`

Candidate = Reference.

Esperado: passa.

### C2 — igualdade em `lte`

Esperado: passa.

### C3 — `gte` acima

Passa.

### C4 — `gte` abaixo

Falha.

### C5 — `lte` abaixo

Passa.

### C6 — `lte` acima

Falha.

### C7 — ignore

Valor candidato não importa para aprovação e não atualiza referência.

### C8 — mutationChance

Testar `gte`, `lte`, `ignore` exatamente como qualquer outro trait.

---

# 56. Teste crítico de rejeição sem atualização

Setup:

```text
Reference:
Production = 100
Toughness = 100

Modes:
Production = gte
Toughness = gte

Candidate:
Production = 500
Toughness = 99
```

Esperado:

```text
Rejected
Production reference continua 100
Toughness reference continua 100
```

Se Production virar 500, o build deve ser considerado incorreto.

---

# 57. Teste crítico de ignore congelado

Setup:

```text
Lifespan reference = 700
Lifespan mode = ignore
```

Aprovar candidata com:

```text
Lifespan = 200
```

Esperado:

```text
reference continua 700
```

Depois alterar para:

```text
Lifespan = gte
```

A próxima candidata deve ser comparada contra 700.

---

# 58. Testes de linhagem

### L1 — mesma family + mesmo subtype

Pode seguir para genética.

### L2 — family diferente

Rejected incompatible subspecies.

### L3 — mesma family + subtype diferente

Rejected incompatible subspecies.

### L4 — Queen vs Young Queen, mesma linhagem

Devem ser compatíveis.

---

# 59. Testes de outputs/backpressure

### O1 — Approved ocupado

Input pode ter qualquer item.

Esperado:

- WAITING_OUTPUTS;
- não validar Input;
- não mexer em genome;
- não mover item.

### O2 — Rejected ocupado

Mesmo comportamento.

### O3 — ambos ocupados

Mesmo comportamento.

### O4 — output possui item stackável

Mesmo assim bloqueia.

---

# 60. Testes de stack

Input com stack > 1.

Esperado:

- processar somente 1 unidade;
- unidade vai ao output;
- restante permanece no Input;
- como output está ocupado, próximo ciclo para.

---

# 61. Testes de Reset

### R1 — perfil ativo, slots vazios

Clique Reset.

Esperado:

- profile nil;
- references nil;
- modes preservados;
- lastResult NONE.

### R2 — Input ocupado

Reset não move/remove item.

Quando outputs permitirem processamento, item válido vira baseline.

### R3 — Approved/Rejeted ocupado

Reset permitido da mesma forma.

### R4 — modos personalizados

Reset não altera os modos.

---

# 62. Testes de persistência

### P1 — salvar/recarregar com objeto colocado

Esperado:

- profile permanece;
- references permanecem;
- modes permanecem.

### P2 — quebrar/recolocar

É aceitável perder perfil/referências/modos.

Não exigir preservação no item.

---

# 63. Testes de ITD

### I1 — ITD insere no offset 0

Máquina processa normalmente.

### I2 — ITD remove offset 1

Approved é esvaziado e libera próximo ciclo quando Rejected também vazio.

### I3 — ITD remove offset 2

Mesmo conceito.

### I4 — ITD mal configurado insere em Approved

Máquina entra WAITING_OUTPUTS.

Não tentar “corrigir” configuração do jogador.

### I5 — ITD retira Input antes do ciclo

Próximo tick vê Input vazio.

Sem inconsistência.

---

# 64. Testes de multiplayer

### M1 — dois jogadores abrem a interface

Ambos veem os mesmos 3 slots.

### M2 — A muda modo

B vê alteração após refresh.

### M3 — A usa Reset

B vê profile/references limpos após refresh.

### M4 — A retira item do output

B deve refletir container atualizado nativamente.

### M5 — ITD altera container com duas GUIs abertas

Ambas devem acompanhar estado real.

### M6 — comandos de modo quase simultâneos

Último aplicado prevalece; sem lock/conflict UI.

---

# 65. Testes de atomicidade

### T1 — Input muda antes da retirada

Analisar A, trocar por B antes do commit.

Esperado:

- abortar;
- não aplicar resultado de A a B;
- nenhuma referência muda.

### T2 — output deixa de estar vazio

Abortar antes de mover.

### T3 — retirada falha

Nenhuma referência muda.

### T4 — inserção no output falha

- rollback ao Input;
- nenhuma referência muda.

---

# 66. Testes da GUI

### G1 — sem perfil

Mostrar `—` para family/subspecies/references.

### G2 — baseline criada

Mostrar valores display corretos.

### G3 — raw/display mutationChance

Confirmar que GUI mostra valor convertido e Machine usa raw.

### G4 — radio groups

Somente um modo ativo por trait.

### G5 — Reset instantâneo

Sem confirmation modal.

### G6 — GUI fechada

Processamento continua.

---

# 67. Critérios de release do MVP

Release bloqueado até que todos estes pontos sejam verdadeiros:

1. genética extraída corretamente de Queen e Young Queen reais do FU;
2. rejected nunca altera referência;
3. baseline funciona;
4. `gte/lte/ignore` funcionam nos seis traits;
5. mutationChance não tem tratamento especial;
6. family + subtype definem compatibilidade;
7. outputs ocupados impedem qualquer processamento/validação;
8. uma unidade por ciclo;
9. máquina funciona com GUI fechada;
10. container real de 3 slots funciona para jogador e ITD;
11. multiplayer compartilha o mesmo estado;
12. Reset preserva modos/slots;
13. crafting/desbloqueio funcionam;
14. máquina não consome energia;
15. sprite físico permanece estático.

---

# 68. Decisões que NÃO devem ser reinterpretadas

## MUST

- usar OpenStarbound como baseline técnico;
- manter container real de 3 slots;
- usar interface de container scriptável;
- Input lógico = slot 1 / offset 0;
- Approved lógico = slot 2 / offset 1;
- Rejected lógico = slot 3 / offset 2;
- ambos outputs precisam estar vazios antes de qualquer validação;
- processar apenas 1 unidade por ciclo;
- somente Queen/Young Queen analisadas são candidatas;
- usar tags do FU;
- identidade = `family + subtype`;
- armazenar raw;
- exibir display;
- exatamente seis traits;
- rejected nunca altera refs;
- approved atualiza apenas traits ativos;
- ignore congela ref;
- primeira candidata válida cria baseline e vai para Approved;
- Reset imediato sem confirmação;
- Reset preserva modos e slots;
- object script é autoridade;
- GUI funciona como view/controle;
- ITD vê storage comum de 3 slots;
- sem consumo de energia;
- sprite estático.

## MUST NOT

- usar subtype isoladamente como identidade;
- usar nome textual/substring para determinar Queen em vez de tags;
- tratar `wild` como sinônimo de não analisado;
- processar drones;
- usar `parameters.lifespan` como gene `queenLifespan`;
- gerar genome ausente;
- reparar genome inválido;
- comparar valores display;
- incluir `workTime`;
- incluir `miteResistance`;
- inverter automaticamente mutationChance;
- aproveitar parcialmente traits de rejected;
- criar score/pesos/média;
- atualizar ref antes de colocar item em Approved;
- criar fila interna;
- criar buffer oculto;
- criar semântica especial de I/O para ITD;
- criar locks multiplayer próprios;
- usar ScriptPane com slots proxy;
- serializar perfil no item dropado no MVP;
- adicionar animação/light effects ao objeto.

---

# 69. Pontos que PODEM ser ajustados sem mudar design

Somente se testes indicarem necessidade:

- polling de 1,0 s pode ser levemente ajustado;
- refresh de GUI ~0,5 s pode ser ajustado;
- posicionamento pixel-perfect dos widgets;
- número exato de anchors de placement;
- shape fino da collision/spaceScan;
- nomes físicos dos arquivos, desde que IDs e namespace permaneçam consistentes;
- detalhes artísticos do sprite mantendo as regras visuais.

Qualquer alteração de lógica genética, slots, energia, baseline, atualização de referências, ITD ou tipo de interface é mudança de design e não deve ser feita silenciosamente.

---

# 70. Checklist de auditoria antes do primeiro commit funcional

Antes de implementar lógica, confirmar no checkout local atual:

- [ ] caminho e API atuais de `/bees/genomeLibrary.lua`;
- [ ] `statOrder` ainda possui os 9 campos esperados;
- [ ] `beeData.config` mantém `stats[family][subtype]`;
- [ ] tags `bee`, `queen`, `youngQueen` continuam válidas;
- [ ] `genomeInspected` continua sendo o indicador de análise;
- [ ] `transferUtil.loadSelfContainer()` continua disponível;
- [ ] Bee Station ainda usa filtro `beestation`;
- [ ] aba `functional` ainda existe;
- [ ] `requiresBlueprint` ainda está true;
- [ ] identificar IDs exatos dos 4 materiais da recipe;
- [ ] confirmar sintaxe OpenStarbound atual para script de ContainerPane;
- [ ] confirmar método atual para obter entity id do container na interface;
- [ ] confirmar callbacks/radio groups disponíveis no runtime;
- [ ] confirmar que `itemgrid`/`slotOffset` funcionam na interface scriptável escolhida.

Se algum ponto mudou no runtime/FU, adaptar a integração **sem mudar a semântica funcional definida neste documento**.

---

# 71. Erros comuns que o implementador deve evitar

## Erro 1 — atualizar refs trait a trait antes de saber se a candidata inteira passou

Errado:

```text
Production passou → atualizar Production
Toughness falhou → rejeitar
```

Correto:

```text
avaliar tudo
↓
se qualquer um falhar → rejected, refs intactas
↓
se todos passarem → mover para Approved
↓
commit de refs ativas
```

## Erro 2 — usar display em comparação

Não comparar porcentagens/valores formatados.

Sempre raw.

## Erro 3 — `ignore` apagar referência

`ignore` congela, não zera.

## Erro 4 — baseline respeitar ignore e deixar referência nil

Baseline sempre armazena os seis traits.

## Erro 5 — tratar output com stack compatível como disponível

Qualquer item no output bloqueia.

## Erro 6 — validar Input enquanto output ocupado

Não deve nem chamar Adapter nesse estado.

## Erro 7 — confundir Queen lifespan atual com gene

`parameters.lifespan` ≠ `queenLifespan` genético.

## Erro 8 — identificar family/subtype por tooltip

Usar dados/estrutura, não texto de UI.

## Erro 9 — simular slots em ScriptPane

Os três slots devem ser container real.

## Erro 10 — fazer GUI decidir aprovação

Processamento pertence ao object script.

---

# 72. Prompt recomendado para iniciar o chat de implementação

Copiar o documento inteiro para o novo chat e iniciar com algo equivalente a:

> Você está implementando um addon de Starbound para Frackin' Universe chamado FUBeesGenetics, usando OpenStarbound como runtime de referência. O documento anexado é a especificação autoritativa e autossuficiente do MVP Genetic Queen Filter. Não assuma contexto além dele e não reinterprete decisões marcadas como MUST/MUST NOT. Comece apenas pelas Fases 1 e 2: estrutura mínima do addon e Bee Genetic Adapter. Antes de criar arquivos, audite os paths/APIs atuais do FU citados no documento e reporte qualquer divergência. Não implemente ainda Machine Object completo, GUI, crafting ou arte. Para cada arquivo criado, explique o que foi feito, como funciona e por que segue a especificação. Execute validações incrementais e não avance de fase enquanto os critérios da fase atual não estiverem satisfeitos.

---

# 73. Referências públicas auditadas em 2026-10-06

Estas URLs são referências técnicas e devem ser rechecadas durante implementação caso o repositório tenha mudado.

## Frackin' Universe

- Repository: https://github.com/sayterdarkwynd/FrackinUniverse
- Genome library: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/bees/genomeLibrary.lua
- Bee data: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/bees/beeData.config
- Bee builder: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/bees/beeBuilder.lua
- Apiary: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/bees/apiary.lua
- Transfer utility: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/scripts/kheAA/transferUtil.lua
- Bee Station object: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/objects/bees/beestation/beestation.object
- Bee Station UI: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/interface/windowconfig/beestation.config
- Normal Apiary recipe example: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/recipes/bees/normalapiary.recipe
- Lab Centrifuge object: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/objects/power/centrifuge/centrifuge.object
- Industrial centrifuge UI: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/interface/bees/industrialcentrifuge/industrialcentrifuge2.config
- Centrifuge processing script: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/objects/bees/centrifuge.lua
- Resource generator / ITD update pattern: https://github.com/sayterdarkwynd/FrackinUniverse/blob/master/objects/power/isn_atmoscondenser/isn_resource_generator.lua

## OpenStarbound

- Repository: https://github.com/OpenStarbound/OpenStarbound
- Lua world API: https://github.com/OpenStarbound/OpenStarbound/blob/main/doc/lua/world.md

## Cross-reference útil para container-interface scripting

O ecossistema xStarbound documenta explicitamente `container interface scripts` e indica compatibilidade/overlap de APIs com OpenStarbound em várias áreas. Esta referência pode ajudar a localizar o equivalente no OpenStarbound atual, mas não deve substituir verificação no runtime alvo:

- https://github.com/xStarbound/xStarbound/blob/main/doc/lua/world.md

---

# 74. Estado final da especificação

O MVP está funcionalmente fechado.

A implementação não precisa tomar novas decisões de produto sobre:

- propósito da máquina;
- elegibilidade;
- genética;
- comparadores;
- baseline;
- atualização de referências;
- Reset;
- slots;
- ITD;
- energia;
- multiplayer;
- crafting;
- desbloqueio;
- footprint;
- animação;
- textos raciais;
- namespace.

Questões remanescentes são de **mapeamento técnico e execução** (por exemplo IDs exatos de materiais e sintaxe atual de script de ContainerPane), não de design.

