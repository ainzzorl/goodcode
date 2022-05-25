---
title: "Leela Chess Zero - Chess Board Representation [C++]"
layout: default
last_modified_date: 2021-11-30T20:18:00+0300 # TODO
nav_order: 21

status: PUBLISHED
language: C++
short-title: Chess Board Representation
project:
  name: Leela Chess Zero
  key: lc0
  home-page: https://github.com/LeelaChessZero/lc0
tags: [chess]
---

{% include article-meta.html article=page %}

## Context

Leela Chess Zero (abbreviated as LCZero, lc0) is a chess engine designed to play chess via neural network, specifically those of the [LeelaChessZero](https://lczero.org/) project. Lc0 is considered one of the strongest chess engines in the world.

The reader is expected to know the rules of chess.

## Problem

A chess engine needs to represent the chess board. Board representation is fundamental to all aspects of a chess program including move generation, the evaluation function, and making and unmaking moves (i.e. search) as well as maintaining the state of the game during play.

In this article we only focus on the representation, leaving aside other aspects of the engine such as evaluation, search, etc.

## Overview

Lc0 has the [`ChessBoard`](https://github.com/LeelaChessZero/lc0/blob/4d89f0870dfe3a33cac36c4d9f2850fcb4e0c179/src/chess/board.h#L60-L272) class representing a chess position. it makes heavy use of [BitBoards](https://www.chessprogramming.org/Bitboards) to store piece positions. It defines methods like `GeneratePseudolegalMoves`, `ApplyMove`, `IsUnderAttack`.

Some terms necessary to understand the code:

- Bitboard ([Chess Programming](https://www.chessprogramming.org/Bitboards), [Wikipedia](https://en.wikipedia.org/wiki/Bitboard)) - bit array data structure, where each bit corresponds to a game board space. This allows parallel bitwise operations to set or query the game state, or determine moves or plays in the game. It's often abbreviated as _BB_ in the code.
- [Ply](<https://en.wikipedia.org/wiki/Ply_(game_theory)>) - one turn taken by one side.
- [50 move rule](https://en.wikipedia.org/wiki/Fifty-move_rule) - states that a player can claim a draw if no capture has been made and no pawn has been moved in the last fifty moves.
- [FEN](https://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation) - Forsyth–Edwards Notation (FEN) is a standard notation for describing a particular board position of a chess game.
- [Pseudo-legal move](https://www.chessprogramming.org/Pseudo-Legal_Move) - is legal in the sense that it is consistent with the current board representation it is assigned to, but may still be illegal if they leave the own king in check.

## Implementation details

_Dive deeper into the implementation. Include code snippets. Insert them as-is, except remove irrelevant details when necessary. Add permalinks to everything._

### [Declaration](https://github.com/LeelaChessZero/lc0/blob/4d89f0870dfe3a33cac36c4d9f2850fcb4e0c179/src/chess/board.h#L60-L272)

There are, among other, methods to:

[set position from FEN](https://github.com/LeelaChessZero/lc0/blob/4d89f0870dfe3a33cac36c4d9f2850fcb4e0c179/src/chess/board.h#L73-L74)

```c++
void SetFromFen(std::string fen, int* rule50_ply = nullptr,
                int* moves = nullptr);
```

Generate and make moves, check if a move is legal:

```c++
// Generates list of possible moves for "ours" (white), but may leave king
// under check.
MoveList GeneratePseudolegalMoves() const;
// Applies the move. (Only for "ours" (white)). Returns true if 50 moves
// counter should be removed.
bool ApplyMove(Move move);
// Checks if the square is under attack from "theirs" (black).
bool IsUnderAttack(BoardSquare square) const;
// Generates the king attack info used for legal move detection.
KingAttackInfo GenerateKingAttackInfo() const;
// Checks if "our" (white) king is under check.
bool IsUnderCheck() const { return IsUnderAttack(our_king_); }

// Checks whether at least one of the sides has mating material.
bool HasMatingMaterial() const;
// Generates legal moves.
MoveList GenerateLegalMoves() const;
// Check whether pseudolegal move is legal.
bool IsLegalMove(Move move, const KingAttackInfo& king_attack_info) const;
// Returns whether two moves are actually the same move in the position.
bool IsSameMove(Move move1, Move move2) const;
// Returns the same move but with castling encoded in legacy way.
Move GetLegacyMove(Move move) const;
// Returns the same move but with castling encoded in modern way.
Move GetModernMove(Move move) const;
```

Let's look into how some of these methods are implemented.

TODO: bitboard

## Testing

_Describe how it's tested. Include snippets and links._

## Observations

_Add more observations about the code. Optional. Consider writing observations in relevant sections of "Implementation details"._

## Related

_Discuss similar and/or related implementations. Optional._

## References

_Add references to relevant documents._

## Copyright notice

_What license is the project under._

_Who owns the copyright._
