---
title:  "Stockfish - Chess Board Representation [C++]"
layout: default
last_modified_date: 2021-08-27T14:46:00+0300
nav_order: 6

status: PUBLISHED
language: C++
project:
    name: Stockfish
    key: stockfish
    home-page: https://github.com/official-stockfish/Stockfish
tags: [chess, bitboard]
---

{% include article-meta.html article=page %}

## Context

Stockfish is one of the most famous and most powerful chess engines. Stockfish is consistently ranked first or near the top of most chess-engine rating lists and is the strongest CPU chess engine in the world.

## Problem

A chess engine needs to represent the chess board. Board representation is fundamental to all aspects of a chess program including move generation, the evaluation function, and making and unmaking moves (i.e. search) as well as maintaining the state of the game during play.

## Overview

Stockfish has the [`Position`](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.h#L74-L202) class representing a chess position. It makes use of various clever, while standard, data structures and techniques, such as [BitBoards](https://www.chessprogramming.org/Bitboards).

## Implementation details

Rather than try to explain the implementation, we will just explain some terminology and encourage you to read the code:
* Bitboard ([Chess Programming](https://www.chessprogramming.org/Bitboards), [Wikipedia](https://en.wikipedia.org/wiki/Bitboard)) - bit array data structure, where each bit corresponds to a game board space. This allows parallel bitwise operations to set or query the game state, or determine moves or plays in the game.
* [Ply](https://en.wikipedia.org/wiki/Ply_(game_theory)) - one turn taken by one side.
* [50 move rule](https://en.wikipedia.org/wiki/Fifty-move_rule) - states that a player can claim a draw if no capture has been made and no pawn has been moved in the last fifty moves.
* [FEN](https://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation) - Forsyth–Edwards Notation (FEN) is a standard notation for describing a particular board position of a chess game.
* [Pseudo-legal move](https://www.chessprogramming.org/Pseudo-Legal_Move) - is legal in the sense that it is consistent with the current board representation it is assigned to, but may still be illegal if they leave the own king in check.

[Declaration](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.h#L74-L202):

```c++
/// Position class stores information regarding the board representation as
/// pieces, side to move, hash keys, castling info, etc. Important methods are
/// do_move() and undo_move(), used by the search to update node info when
/// traversing the search tree.
class Thread;

class Position {
public:
  static void init();

  Position() = default;
  Position(const Position&) = delete;
  Position& operator=(const Position&) = delete;

  // FEN string input/output
  Position& set(const std::string& fenStr, bool isChess960, StateInfo* si, Thread* th);
  Position& set(const std::string& code, Color c, StateInfo* si);
  std::string fen() const;

  // Position representation
  Bitboard pieces(PieceType pt) const;
  Bitboard pieces(PieceType pt1, PieceType pt2) const;
  Bitboard pieces(Color c) const;
  Bitboard pieces(Color c, PieceType pt) const;
  Bitboard pieces(Color c, PieceType pt1, PieceType pt2) const;
  Piece piece_on(Square s) const;
  Square ep_square() const;
  bool empty(Square s) const;
  template<PieceType Pt> int count(Color c) const;
  template<PieceType Pt> int count() const;
  template<PieceType Pt> Square square(Color c) const;
  bool is_on_semiopen_file(Color c, Square s) const;

  // Castling
  CastlingRights castling_rights(Color c) const;
  bool can_castle(CastlingRights cr) const;
  bool castling_impeded(CastlingRights cr) const;
  Square castling_rook_square(CastlingRights cr) const;

  // Checking
  Bitboard checkers() const;
  Bitboard blockers_for_king(Color c) const;
  Bitboard check_squares(PieceType pt) const;
  Bitboard pinners(Color c) const;

  // Attacks to/from a given square
  Bitboard attackers_to(Square s) const;
  Bitboard attackers_to(Square s, Bitboard occupied) const;
  Bitboard slider_blockers(Bitboard sliders, Square s, Bitboard& pinners) const;

  // Properties of moves
  bool legal(Move m) const;
  bool pseudo_legal(const Move m) const;
  bool capture(Move m) const;
  bool capture_or_promotion(Move m) const;
  bool gives_check(Move m) const;
  Piece moved_piece(Move m) const;
  Piece captured_piece() const;

  // Piece specific
  bool pawn_passed(Color c, Square s) const;
  bool opposite_bishops() const;
  int  pawns_on_same_color_squares(Color c, Square s) const;

  // Doing and undoing moves
  void do_move(Move m, StateInfo& newSt);
  void do_move(Move m, StateInfo& newSt, bool givesCheck);
  void undo_move(Move m);
  void do_null_move(StateInfo& newSt);
  void undo_null_move();

  // Static Exchange Evaluation
  bool see_ge(Move m, Value threshold = VALUE_ZERO) const;

  // Accessing hash keys
  Key key() const;
  Key key_after(Move m) const;
  Key material_key() const;
  Key pawn_key() const;

  // Other properties of the position
  Color side_to_move() const;
  int game_ply() const;
  bool is_chess960() const;
  Thread* this_thread() const;
  bool is_draw(int ply) const;
  bool has_game_cycle(int ply) const;
  bool has_repeated() const;
  int rule50_count() const;
  Score psq_score() const;
  Value non_pawn_material(Color c) const;
  Value non_pawn_material() const;

  // Position consistency check, for debugging
  bool pos_is_ok() const;
  void flip();

  // Used by NNUE
  StateInfo* state() const;

  void put_piece(Piece pc, Square s);
  void remove_piece(Square s);

private:
  // Initialization helpers (used while setting up a position)
  void set_castling_right(Color c, Square rfrom);
  void set_state(StateInfo* si) const;
  void set_check_info(StateInfo* si) const;

  // Other helpers
  void move_piece(Square from, Square to);
  template<bool Do>
  void do_castling(Color us, Square from, Square& to, Square& rfrom, Square& rto);

  // Data members
  Piece board[SQUARE_NB];
  Bitboard byTypeBB[PIECE_TYPE_NB];
  Bitboard byColorBB[COLOR_NB];
  int pieceCount[PIECE_NB];
  int castlingRightsMask[SQUARE_NB];
  Square castlingRookSquare[CASTLING_RIGHT_NB];
  Bitboard castlingPath[CASTLING_RIGHT_NB];
  Thread* thisThread;
  StateInfo* st;
  int gamePly;
  Color sideToMove;
  Score psq;
  bool chess960;
};
```

[Putting a piece](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.h#L374-L382):
```c++
inline void Position::put_piece(Piece pc, Square s) {

  board[s] = pc;
  byTypeBB[ALL_PIECES] |= byTypeBB[type_of(pc)] |= s;
  byColorBB[color_of(pc)] |= s;
  pieceCount[pc]++;
  pieceCount[make_piece(color_of(pc), ALL_PIECES)]++;
  psq += PSQT::psq[pc][s];
}
```

Implementation: [position.cpp](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.cpp).

The key method - [making a move](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.cpp#L679-L893):
```c++
/// Position::do_move() makes a move, and saves all information necessary
/// to a StateInfo object. The move is assumed to be legal. Pseudo-legal
/// moves should be filtered out before this function is called.

void Position::do_move(Move m, StateInfo& newSt, bool givesCheck) {

  assert(is_ok(m));
  assert(&newSt != st);

  thisThread->nodes.fetch_add(1, std::memory_order_relaxed);
  Key k = st->key ^ Zobrist::side;

  // Copy some fields of the old state to our new StateInfo object except the
  // ones which are going to be recalculated from scratch anyway and then switch
  // our state pointer to point to the new (ready to be updated) state.
  std::memcpy(&newSt, st, offsetof(StateInfo, key));
  newSt.previous = st;
  st = &newSt;

  // Increment ply counters. In particular, rule50 will be reset to zero later on
  // in case of a capture or a pawn move.
  ++gamePly;
  ++st->rule50;
  ++st->pliesFromNull;

  // Used by NNUE
  st->accumulator.computed[WHITE] = false;
  st->accumulator.computed[BLACK] = false;
  auto& dp = st->dirtyPiece;
  dp.dirty_num = 1;

  Color us = sideToMove;
  Color them = ~us;
  Square from = from_sq(m);
  Square to = to_sq(m);
  Piece pc = piece_on(from);
  Piece captured = type_of(m) == EN_PASSANT ? make_piece(them, PAWN) : piece_on(to);

  assert(color_of(pc) == us);
  assert(captured == NO_PIECE || color_of(captured) == (type_of(m) != CASTLING ? them : us));
  assert(type_of(captured) != KING);

  if (type_of(m) == CASTLING)
  {
      assert(pc == make_piece(us, KING));
      assert(captured == make_piece(us, ROOK));

      Square rfrom, rto;
      do_castling<true>(us, from, to, rfrom, rto);

      k ^= Zobrist::psq[captured][rfrom] ^ Zobrist::psq[captured][rto];
      captured = NO_PIECE;
  }

  if (captured)
  {
      Square capsq = to;

      // If the captured piece is a pawn, update pawn hash key, otherwise
      // update non-pawn material.
      if (type_of(captured) == PAWN)
      {
          if (type_of(m) == EN_PASSANT)
          {
              capsq -= pawn_push(us);

              assert(pc == make_piece(us, PAWN));
              assert(to == st->epSquare);
              assert(relative_rank(us, to) == RANK_6);
              assert(piece_on(to) == NO_PIECE);
              assert(piece_on(capsq) == make_piece(them, PAWN));
          }

          st->pawnKey ^= Zobrist::psq[captured][capsq];
      }
      else
          st->nonPawnMaterial[them] -= PieceValue[MG][captured];

      if (Eval::useNNUE)
      {
          dp.dirty_num = 2;  // 1 piece moved, 1 piece captured
          dp.piece[1] = captured;
          dp.from[1] = capsq;
          dp.to[1] = SQ_NONE;
      }

      // Update board and piece lists
      remove_piece(capsq);

      if (type_of(m) == EN_PASSANT)
          board[capsq] = NO_PIECE;

      // Update material hash key and prefetch access to materialTable
      k ^= Zobrist::psq[captured][capsq];
      st->materialKey ^= Zobrist::psq[captured][pieceCount[captured]];
      prefetch(thisThread->materialTable[st->materialKey]);

      // Reset rule 50 counter
      st->rule50 = 0;
  }

  // Update hash key
  k ^= Zobrist::psq[pc][from] ^ Zobrist::psq[pc][to];

  // Reset en passant square
  if (st->epSquare != SQ_NONE)
  {
      k ^= Zobrist::enpassant[file_of(st->epSquare)];
      st->epSquare = SQ_NONE;
  }

  // Update castling rights if needed
  if (st->castlingRights && (castlingRightsMask[from] | castlingRightsMask[to]))
  {
      k ^= Zobrist::castling[st->castlingRights];
      st->castlingRights &= ~(castlingRightsMask[from] | castlingRightsMask[to]);
      k ^= Zobrist::castling[st->castlingRights];
  }

  // Move the piece. The tricky Chess960 castling is handled earlier
  if (type_of(m) != CASTLING)
  {
      if (Eval::useNNUE)
      {
          dp.piece[0] = pc;
          dp.from[0] = from;
          dp.to[0] = to;
      }

      move_piece(from, to);
  }

  // If the moving piece is a pawn do some special extra work
  if (type_of(pc) == PAWN)
  {
      // Set en passant square if the moved pawn can be captured
      if (   (int(to) ^ int(from)) == 16
          && (pawn_attacks_bb(us, to - pawn_push(us)) & pieces(them, PAWN)))
      {
          st->epSquare = to - pawn_push(us);
          k ^= Zobrist::enpassant[file_of(st->epSquare)];
      }

      else if (type_of(m) == PROMOTION)
      {
          Piece promotion = make_piece(us, promotion_type(m));

          assert(relative_rank(us, to) == RANK_8);
          assert(type_of(promotion) >= KNIGHT && type_of(promotion) <= QUEEN);

          remove_piece(to);
          put_piece(promotion, to);

          if (Eval::useNNUE)
          {
              // Promoting pawn to SQ_NONE, promoted piece from SQ_NONE
              dp.to[0] = SQ_NONE;
              dp.piece[dp.dirty_num] = promotion;
              dp.from[dp.dirty_num] = SQ_NONE;
              dp.to[dp.dirty_num] = to;
              dp.dirty_num++;
          }

          // Update hash keys
          k ^= Zobrist::psq[pc][to] ^ Zobrist::psq[promotion][to];
          st->pawnKey ^= Zobrist::psq[pc][to];
          st->materialKey ^=  Zobrist::psq[promotion][pieceCount[promotion]-1]
                            ^ Zobrist::psq[pc][pieceCount[pc]];

          // Update material
          st->nonPawnMaterial[us] += PieceValue[MG][promotion];
      }

      // Update pawn hash key
      st->pawnKey ^= Zobrist::psq[pc][from] ^ Zobrist::psq[pc][to];

      // Reset rule 50 draw counter
      st->rule50 = 0;
  }

  // Set capture piece
  st->capturedPiece = captured;

  // Update the key with the final value
  st->key = k;

  // Calculate checkers bitboard (if move gives check)
  st->checkersBB = givesCheck ? attackers_to(square<KING>(them)) & pieces(us) : 0;

  sideToMove = ~sideToMove;

  // Update king attacks used for fast check detection
  set_check_info(st);

  // Calculate the repetition info. It is the ply distance from the previous
  // occurrence of the same position, negative in the 3-fold case, or zero
  // if the position was not repeated.
  st->repetition = 0;
  int end = std::min(st->rule50, st->pliesFromNull);
  if (end >= 4)
  {
      StateInfo* stp = st->previous->previous;
      for (int i = 4; i <= end; i += 2)
      {
          stp = stp->previous->previous;
          if (stp->key == st->key)
          {
              st->repetition = stp->repetition ? -i : i;
              break;
          }
      }
  }

  assert(pos_is_ok());
}
```

## Testing

Stockfish is tested with [Fishtest](https://github.com/glinscott/fishtest), a distributed task queue for testing chess engines.

## Observations

* A significant part of the code is dedicated to somewhat rare rules of the game: en-passant, 50 move, castling rights.
* A lot of bit arithmetics is going there. E.g. [`if (   (int(to) ^ int(from)) == 16`](https://github.com/official-stockfish/Stockfish/blob/8fc297c50647317185d4c41b3443a0e686412681/src/position.cpp#L815) means that `from` is two rows away from `to`.

## Related

* [Board representation in Lc0 aka Leela](https://github.com/LeelaChessZero/lc0/blob/master/src/chess/board.h), a popular chess engine making heavy use of neural networks.

## References

* [GitHub Repo](https://github.com/official-stockfish/Stockfish)
* [Board Representation on Chess Programming Wiki](https://www.chessprogramming.org/Board_Representation)
* [Board Representation on Wikipedia](https://en.wikipedia.org/wiki/Board_representation_(computer_chess))
* [How Stockfish Works: An Evaluation of the Databases Behind the Top Open-Source Chess Engine](http://rin.io/chess-engine/)

## Copyright notice

Stockfish is licensed under the [GNU General Public License v3.0](https://github.com/official-stockfish/Stockfish/blob/master/Copying.txt).

Copyright (C) 2007 Free Software Foundation, Inc. <http://fsf.org/>
