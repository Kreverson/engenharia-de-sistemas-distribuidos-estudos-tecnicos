# 48 — Low-Level Design: Framework e Exemplos Práticos

> O guia definitivo para entrevistas de design orientado a objetos. Cobrindo o framework de raciocínio, princípios SOLID aplicados, e dois exemplos completos: Connect 4 e Amazon Locker.

---

## 1. O Framework de LLD em Entrevistas

### As 5 etapas em ordem

```
1. Requirements     → Clarificar o que construir (5-7 min)
2. Entities         → Identificar os "substantivos" do sistema (2-3 min)
3. Class Design     → Definir estado e comportamento de cada classe (10-15 min)
4. Implementation   → Código/pseudo-código dos métodos chave (15-20 min)
5. Extensibility    → Responder follow-ups de evolução do sistema (5-10 min)
```

### Por que não pular para o código

Candidatos que pulam para a implementação frequentemente:
- Escolhem abstrações erradas que precisam ser refeitas
- Perdem tempo em implementações que o entrevistador não queria
- Não demonstram capacidade de análise (problem analysis é explicitamente avaliado)

---

## 2. Etapa 1: Requirements — As 3 Categorias

Ao receber um enunciado vago, estruture suas perguntas em 3 grupos:

### Capacidades primárias (o que o sistema faz)
```
"Como os usuários interagem com o sistema?"
"Quais são todos os estados finais possíveis?"
"O que acontece em cada ação principal?"
```

### Error handling (como o sistema falha com elegância)
```
"O que acontece se [ação inválida]? Lança exceção, retorna false, ignora?"
"Preciso validar inputs antes de processar?"
```

### Scope boundaries (o que está fora)
```
"É 1 jogo ou múltiplos simultâneos?"
"Preciso implementar a UI ou apenas a lógica?"
"Há autenticação/rede envolvida?"
```

**Output:** escreva os requisitos finalizados. Isso é o "contrato" da entrevista.

---

## 3. Etapa 2: Entities — Extraindo os Substantivos

```
Técnica: percorra os requisitos procurando SUBSTANTIVOS

"Two players take turns dropping discs into a board"
          ↑                              ↑
        Player                         Board (+ Disc implícito)

"The game ends when someone gets four in a row"
   ↑
  Game

Lista inicial: Player, Board, Disc, Game

Filtragem:
  Disc → provavelmente só um enum (DiscColor), não classe separada ✓
  Player → classe simples (só dados: nome, cor) ✓
  Board → classe com lógica de grid ✓
  Game → orquestrador principal ✓
```

---

## 4. Etapa 3: Class Design — Estado + Comportamento

### O Princípio Guia: Single Responsibility Principle (SRP)

> Cada classe deve ter uma única razão para mudar.

**Aplicação prática:**
```
Game → responsável por: turno, estado do jogo, orquestração
Board → responsável por: grid, posicionamento de discos, detecção de vitória
Player → responsável por: identidade do jogador

Se mudar o tamanho do grid → só Board muda
Se adicionar undo → só Game muda  
Se adicionar IA → só Game muda (ou nova classe Bot)
```

### Template para cada classe

```
Para cada entidade, percorra os requisitos e extraia:

ESTADO (o que a classe precisa "lembrar"):
  → Substantivos relacionados à entidade
  → "O jogo precisa saber quem é o jogador atual" → currentPlayer: Player

COMPORTAMENTO (o que a classe precisa "fazer"):
  → Verbos nos requisitos
  → "Jogadores fazem movimentos" → makeMove(player, column): boolean
```

---

## 5. Exemplo 1 — Connect 4

### Requisitos finais

```
✓ 2 jogadores se alternam colocando discos em grid 7 colunas × 6 linhas
✓ Disco cai para a posição mais baixa disponível na coluna
✓ Jogo termina: 4 em linha (horizontal/vertical/diagonal) ou empate (board cheio)
✓ Movimentos inválidos: coluna cheia, turno errado, após game over
✓ Out of scope: UI, múltiplos jogos simultâneos
```

### Entities

```
Game   → orquestrador (estado + turno)
Board  → grid física (posicionamento + detecção de vitória)
Player → dados do jogador (nome + cor)
enum DiscColor { RED, BLUE }
enum GameState { IN_PROGRESS, RED_WINS, BLUE_WINS, DRAW }
```

### Class Design

```
class Game:
  ESTADO:
    player1: Player
    player2: Player
    currentPlayer: Player
    board: Board
    state: GameState    ← enum (melhor que 3 booleans separados)
    winner: Player?     ← nullable (só preenchido quando state = *_WINS)
  
  COMPORTAMENTO (público):
    Game(player1, player2)
    makeMove(player, column) → boolean
    getCurrentPlayer() → Player
    getGameState() → GameState
    getWinner() → Player?
  
  COMPORTAMENTO (privado):
    switchTurn()

class Board:
  ESTADO:
    rows: int = 6
    cols: int = 7
    grid: DiscColor?[][]    ← 2D array, nullable = vazio
  
  COMPORTAMENTO (público):
    Board(rows?, cols?)      ← construtor extensível
    canPlace(col) → boolean
    placeDisc(col, color) → int   ← retorna a ROW onde caiu (-1 se falhou)
    checkWin(row, col, color) → boolean
    isFull() → boolean
  
  COMPORTAMENTO (privado):
    countInDirection(row, col, dr, dc, color) → int
```

### Por que usar enum GameState em vez de múltiplos booleans?

```java
// Ruim: 8 combinações possíveis, apenas 3 válidas
boolean isOver;
boolean hasWinner;
boolean isDraw;

// Bom: apenas 4 estados válidos, impossível representar estado inválido
enum GameState { IN_PROGRESS, RED_WINS, BLUE_WINS, DRAW }
```

Em linguagens como Rust/Swift/TypeScript, o vencedor pode ser parte do tipo enum:
```rust
enum GameState { InProgress, Draw, Win(Player) }
// Impossível ter Win sem especificar quem ganhou
```

### Implementação: makeMove()

```python
def makeMove(self, player: Player, column: int) -> bool:
    # Edge cases primeiro
    if self.state != GameState.IN_PROGRESS:
        return False  # jogo já acabou
    if player != self.currentPlayer:
        return False  # turno errado
    
    # Core logic
    row = self.board.placeDisc(column, player.color)
    if row == -1:
        return False  # coluna cheia ou inválida
    
    # Verificar vitória
    if self.board.checkWin(row, column, player.color):
        self.state = GameState.RED_WINS if player.color == DiscColor.RED else GameState.BLUE_WINS
        self.winner = player
    elif self.board.isFull():
        self.state = GameState.DRAW
    else:
        self.switchTurn()
    
    return True
```

### Implementação: checkWin() — O método mais interessante

```python
def checkWin(self, row: int, col: int, color: DiscColor) -> bool:
    # 4 direções (cada uma tem direção oposta para varrer)
    directions = [
        (0, 1),   # horizontal
        (1, 0),   # vertical
        (1, 1),   # diagonal \
        (1, -1),  # diagonal /
    ]
    
    for dr, dc in directions:
        count = 1  # conta o disco recém-colocado
        count += self._countInDirection(row, col, dr, dc, color)    # direção positiva
        count += self._countInDirection(row, col, -dr, -dc, color)  # direção oposta
        
        if count >= 4:
            return True
    
    return False

def _countInDirection(self, row, col, dr, dc, color) -> int:
    count = 0
    r, c = row + dr, col + dc
    
    while (0 <= r < self.rows and 
           0 <= c < self.cols and 
           self.grid[r][c] == color):
        count += 1
        r += dr
        c += dc
    
    return count
```

**Por que NÃO usar Strategy Pattern aqui:**
```
Estratégia para cada direção seria overengineering:
- Não há casos de uso reais para trocar as direções
- A lógica é idêntica entre as 4 direções
- Adiciona ~50 linhas de código para zero valor

Candidatos que usam Strategy Pattern aqui para "mostrar que sabem"
frequentemente perdem pontos por overengineering.
```

### Extensibilidade — Perguntas comuns

**"Suporte a board de tamanho variável":**
```
Solução: Board constructor com parâmetros rows/cols opcionais
Sem hardcode de 6 ou 7 em nenhum lugar → já funciona
```

**"Adicionar undo/histórico":**
```
Game adiciona:
  moveHistory: Stack<Move>   (Move = {player, row, col})
  undoLastMove() → boolean

Em makeMove: push para o stack após successo
Em undoLastMove: pop do stack → board.clearCell(row, col) → reverter state → reverter currentPlayer
```

**"Adicionar oponente IA":**
```python
# Abordagem elegante: Bot como componente separado
class BotEngine:
    def chooseMove(self, board: Board) -> int:
        # random, minimax, ou qualquer estratégia
        pass

# Game loop (fora da classe Game):
while game.getState() == GameState.IN_PROGRESS:
    if game.getCurrentPlayer() == human_player:
        col = input("Enter column: ")
    else:
        col = bot_engine.chooseMove(game.board)
    game.makeMove(game.getCurrentPlayer(), col)
```

**Não precisa modificar Game ou Board para adicionar IA!**

---

## 6. Exemplo 2 — Amazon Locker

### Requisitos

```
✓ Driver deposita pacote especificando o tamanho (S/M/L)
✓ Sistema abre compartimento certo e retorna código de acesso
✓ Cliente retira pacote digitando o código
✓ Códigos expiram após 7 dias
✓ Staff pode limpar compartimentos com códigos expirados
✓ Inválidos: compartimento cheio, código errado, código expirado
✓ Out of scope: notificação SMS/email, logística de entrega
```

### Class Design

```
class Locker (orquestrador):
  ESTADO:
    compartments: List[Compartment]
    accessTokens: Map[code → AccessToken]   ← O(1) lookup por código
  
  PÚBLICO:
    depositPackage(size: Size) → String   ← retorna código
    pickup(code: String) → void
    openExpiredCompartments() → void
  
  PRIVADO:
    _getAvailableCompartment(size) → Compartment?
    _generateAccessToken(compartment) → AccessToken

class Compartment:
  ESTADO:
    size: Size             ← enum: SMALL, MEDIUM, LARGE
    status: CompartmentStatus  ← enum: AVAILABLE, OCCUPIED, OUT_OF_SERVICE
  
  PÚBLICO:
    getSize() → Size
    isAvailable() → boolean
    markOccupied() → void
    markFree() → void
    open() → void     ← mecânico, apenas simula

class AccessToken:
  ESTADO:
    code: String
    compartment: Compartment
    expiresAt: DateTime
  
  PÚBLICO:
    getCode() → String
    getCompartment() → Compartment
    isExpired() → boolean
```

### Por que Map[code → AccessToken] e não List[AccessToken]?

```
Lista de 100 compartimentos com poucos tokens:
  Busca por código: O(N) scan linear
  
Map code → token:
  Busca por código: O(1) hash lookup
  
Além da performance, o código é mais legível:
  accessTokens["ABC123"] → AccessToken  ← intenção clara
```

### Implementação: depositPackage()

```python
def depositPackage(self, size: Size) -> str:
    # 1. Encontrar compartimento disponível
    compartment = self._getAvailableCompartment(size)
    if compartment is None:
        raise NoCompartmentAvailableError(f"No {size} compartment available")
    
    # 2. Abrir compartimento
    compartment.open()
    
    # 3. Marcar como ocupado
    compartment.markOccupied()
    
    # 4. Gerar token de acesso
    token = self._generateAccessToken(compartment)
    
    # 5. Armazenar token
    self.accessTokens[token.getCode()] = token
    
    # 6. Retornar código
    return token.getCode()

def _getAvailableCompartment(self, size: Size) -> Optional[Compartment]:
    for c in self.compartments:
        if c.getSize() == size and c.isAvailable():
            return c
    return None  # caller trata o None
```

### Implementação: pickup()

```python
def pickup(self, code: str) -> None:
    # Edge cases
    if not code:
        raise InvalidCodeError("Code cannot be empty")
    
    token = self.accessTokens.get(code)
    if token is None:
        raise InvalidCodeError("Code not found")
    
    if token.isExpired():
        raise ExpiredCodeError("Access code has expired")
    
    # Happy path
    compartment = token.getCompartment()
    compartment.open()          # usuário retira o pacote
    compartment.markFree()      # compartimento disponível novamente
    
    # Manter no map mas não deletar → usuário recebe "expired" não "invalid"
    # Limpeza periódica por batch job separado
```

### Decisão de design interessante: deletar ou manter o token após pickup?

```
Opção A: deletar imediatamente
  → Se usuário tenta usar código novamente: "invalid code"
  → Confuso para usuário

Opção B: manter no map (mas compartimento já free)
  → Se usuário tenta usar código novamente: "already used" (mais descritivo)
  → Limpar periodicamente tokens usados + expirados (batch job)
  → Memória: 100 compartimentos × 1 token = negligível
```

---

## 7. Anti-Padrões em LLD (O que NÃO fazer)

### Anti-padrão 1: Injetar padrões desnecessários

```
❌ Strategy Pattern para 4 direções em Connect 4
❌ Factory Pattern para criar instâncias simples
❌ Observer Pattern quando um callback basta

Regra: introduza um padrão apenas quando há clara necessidade de extensão
```

### Anti-padrão 2: Múltiplos booleans em vez de enum

```java
❌ boolean isOver; boolean hasWinner; boolean isDraw;
✅ enum GameState { IN_PROGRESS, WON, DRAW }
```

### Anti-padrão 3: Classes que sabem demais (violação de SRP)

```
❌ Game verifica se 4 em linha (responsabilidade do Board)
❌ Board sabe quem é o "current player" (responsabilidade do Game)
```

### Anti-padrão 4: Modelar entidades externas desnecessariamente

```
Amazon Locker: pacote é entrada do sistema, não precisa ser classe
Instagram: usuário existe fora do sistema de notificação
```

---

## 8. Pseudo-código vs Código Real

```
Em entrevistas: pseudo-código é padrão e suficiente
  → Não precisa de syntax perfect
  → Foco na lógica e estrutura
  → Python-like é universalmente aceito

Se a empresa pede código real (30-35% dos casos):
  → Verifique com o recrutador com antecedência
  → Prepare implementações em sua linguagem no artigo do Hello Interview
  → Escolha a linguagem que você mais domina
```

---

*Tópicos relacionados: `10_framework_system_design.md`, `concorrencia.md`, pattern design resources*
