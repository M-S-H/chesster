<script setup lang="ts">
import { Chess, Move, Piece, Square } from 'chess.js'
import { shuffle } from 'lodash';
import { computed, onMounted, reactive, ref, render } from 'vue';
import pieceValues from './pieceValues'

let game: Chess
const init = ref(false)
const isGameOver = ref(false)
const playerColor: PlayerColor = 'b'
type PlayerColor = 'w' | 'b'
// const pieceValues = {
//   'p': 10,
//   'b': 30,
//   'n': 30,
//   'r': 50,
//   'q': 90,
//   'k': 900
// }
const maxDepth = 3

const pieceIcons = {
  'p': 'pawn',
  'b': 'bishop',
  'n': 'knight',
  'r': 'rook',
  'q': 'queen',
  'k': 'king'
}


const currentPlayer = ref('w' as PlayerColor)
let board = reactive({} as any)
let lastMove = reactive({} as Move)
let selectedPiece = ref(null as any)
const selectedPieceMoves = ref([] as Move[])
const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'].reverse()
const rows = ['1', '2', '3', '4', '5', '6', '7', '8']

const isYourTurn = computed((): boolean => {
  return currentPlayer.value === playerColor
})

function compTurn () {
  const pm = possibleMoves(currentPlayer.value)
  // const move = pm[Math.floor(Math.random() * pm.length)]
  const move = miniMaxMove(currentPlayer.value)
  game.move(move)
  swapPlayer()
  setBoard()
  isGameOver.value = game.isCheckmate() || game.isStalemate() || game.isDraw()
}

function swapPlayer () {
  if (currentPlayer.value == 'w') {
    currentPlayer.value = 'b'
  } else {
    currentPlayer.value = 'w'
  }
}

function nextPlayer (currentPlayer: PlayerColor): PlayerColor {
  if (currentPlayer === 'w') {
    return 'b'
  } else {
    return 'w'
  }
}

function setSelectedPiece (file: string, row: string, piece: any) {
  // Object.assign(selectedPiece, piece)
  selectedPiece.value = {
    piece: piece.piece,
    location: file + row
  }
  // console.log(`${file}${row}: ${piece}`)
  selectedPieceMoves.value = game.moves({ verbose: true, square: (file + row) as Square }) as Move[]
  
  console.log(selectedPieceMoves.value)
  setBoard()
}

function tryMovePiece (file: string, row: string) {
  const pos = file + row
  if (!board[file][row].hasPossibleMoves) return
  if (pos === selectedPiece.value.location) return

  const move = selectedPieceMoves.value.find(m => {
    return m.from === selectedPiece.value.location && m.to === pos
  })
  
  if (!move) return

  game.move(move)
  selectedPiece.value = null
  selectedPieceMoves.value = []
  console.log(evaluateValue(game, playerColor))
  swapPlayer()
  setBoard()
  setTimeout(() => {
    compTurn()
  }, 0);
}

function miniMaxMove(playerToMaximize: PlayerColor): Move {
  let bestMove = {} as Move
  let bestVal = -Infinity
  // const isMaximizingPlayer = true
  const moves = shuffle(game.moves({ verbose: true })) as Move[]
  moves.forEach(m => {
    const tempGame = new Chess(game.fen())
    tempGame.move(m)
    let val = minimax(tempGame, maxDepth - 1, -Infinity, Infinity, playerToMaximize)
    if (val >= bestVal) {
      bestVal = val
      bestMove = m as Move
    }
  })

  console.log(`best: ${bestVal}`)

  return bestMove
}

function minimax(tempGame: Chess, depth: number, alpha: number, beta: number, playerToMaximize: PlayerColor): number {
  if (depth === 0) {
    return evaluateValue(tempGame, playerToMaximize)
  }
  const isMaximizingPlayer = playerToMaximize === tempGame.turn()

  let bestValue = isMaximizingPlayer ? -Infinity : Infinity
  let newAlpha = alpha
  let newBeta = beta
  const moves = tempGame.moves({ verbose: true })


  moves.forEach(m => {
    const newTempGame = new Chess(tempGame.fen())
    newTempGame.move(m)
    const val = minimax(newTempGame, depth - 1, newAlpha, newBeta, playerToMaximize)
    
    if (isMaximizingPlayer) {
      bestValue = Math.max(bestValue, val)
      newAlpha = Math.max(newAlpha, bestValue)
      if (newBeta <= newAlpha) {
        // console.log('early!')
        return bestValue
      }
    } else {
      bestValue = Math.min(bestValue, val)
      newBeta = Math.min(newBeta, bestValue)
      if (newBeta <= newAlpha) {
        // console.log('early!')
        return bestValue
      }
    }
  }) 

  return bestValue
}

// function minimax (gb: Chess, d: number, player: PlayerColor) {
//   if (d === 0 || gb.isGameOver()) {
//     return evaluateValue(gb, player)
//   }

//   let bestVal = player === currentPlayer.value ? -Infinity : Infinity

//   game.moves({ verbose: true }).forEach(m => {
//     const tempGame = new Chess(gb.fen())
//     tempGame.move(m)
//   })
// }

function evaluateValue (gb: Chess, playerToMaximize: PlayerColor): number {
  let val = 0
  let i = 0
  let j = 0
  files.forEach(f => {
    rows.forEach(r => {
      const pos = f + r
      const p = gb.get(pos as Square)
      if (p) {
        let pieceValue = pieceValues[p.type][i][ p.color === 'b' ? 7 - j : j ]
        val += pieceValue * (playerToMaximize === p.color ? 1 : -1)
      }
      j += 1
    })
    i += 1
    j = 0
  })

  return val
}

function setBoard (): any {
  const boardObj = {} as any
  files.forEach(f => {
    boardObj[f] = {}
    rows.forEach(r => {
      const pos = f + r
      const piece = game.get(pos as Square)
      const ss = {
        piece: piece,
        hasPossibleMoves: false,
        hasMoves: game.moves({ square: pos as Square })
      }
      boardObj[f][r] = ss
    })
  })

  selectedPieceMoves.value.forEach(sq => {
    // sq.from[]
    const f = sq.to[0]
    const r = sq.to[1]
    boardObj[f][r].hasPossibleMoves = true
  })

  // if (selectedPiece.value) {
  //   console.log(selectedPiece.value)

  // }

  Object.assign(board, boardObj)
}



function possibleMoves (player: PlayerColor): Move[] {
  const allMoves = game.moves({ verbose: true }) as Move[]
  return allMoves.filter(m => {
    return m.color === player
  })
}

function squareColor(i: number, j: number): 'dark' | 'light' {
  if (i % 2 == 0 && j % 2 == 0) {
    return 'light'
  } else if (i % 2 == 0 && j % 2 != 0) {
    return 'dark'
  } else if (i % 2 != 0 && j % 2 != 0) {
    return 'light'
  } else { // (i % 2 == 0 && j % 2 == 0)
    return 'dark'
  }
}

onMounted(() => {
  game = new Chess()
  setBoard()
  init.value = true;
  (window as any)['board'] = board;
  (window as any)['game'] = game;
  compTurn()
})
</script>s

<template>
  <div class="main">
    <div>{{ isGameOver ? 'Game Over' : '' }}</div>
    <!-- {{ isGameOver }}
    {{ selectedPiece }}
    {{ selectedPieceMoves }} -->
    <!-- <button @click="takeTurn">Take Turn</button> -->
    <div class="board" v-if="init">
      <div
        class="row"
        v-for="(r, i) in rows"
      >
        <div
          v-for="(f, j) in files"
          class="square"
          :class="[squareColor(i,j), { 'can-move': board[f][r].hasPossibleMoves }]"
          @click="tryMovePiece(f,r)"
        >
          <div
            v-if="board[f][r].piece"
            class="piece"
            :class="[board[f][r].piece.color]"
            @click="setSelectedPiece(f, r, board[f][r])"
          >
            <i :class="'fa-solid fa-chess-' + pieceIcons[board[f][r].piece.type]"></i>
            <!-- {{ board[f][r].piece.type }} -->
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
@use "sass:math";
@import "@fortawesome/fontawesome-free/css/all.css";

$squareSize: 60px;
$pieceSize: math.div($squareSize, 2);
$pieceMargin: math.div($squareSize - $pieceSize, 2);

.board {
  width: $squareSize * 8;
  .row {
    display: flex;
    .square {
      width: $squareSize;
      height: $squareSize;
      position: relative;
      
      &.dark {
        background: #999;
      }

      &.light {
        background: #ccc;
      }

      &.can-move::before {
        content: ' ';
        position: absolute;
        top: 0;
        left: 0;
        width: $squareSize;
        height: $squareSize;
        background: rgba(yellow, 0.3);
      }
    }
  }

  .piece {
    width: $pieceSize;
    height: $pieceSize;
    margin-left: $pieceMargin;
    margin-top: $pieceMargin;
    line-height: $pieceSize;
    border-radius: 50%;
    text-align: center;
    font-size: 26px;
    
    &.b {
      // background-color: black;
      color: black;
    }

    &.w {
      // background-color: white;
      color: white;
    }
  }
}
</style>
