---
author: "Sean Leishman"
title: "TurtleChess: Adventures in Chess Engines"
date: "2023-06-24"
tags: ["Project", "Chess", "Python", "Pygame", "ML"]
ShowBreadCrumbs: true
ShowToc: false
weight: 1
---

> Source code can be found at {{< newtabref href="https://github.com/Sean-Leishman/turtle-chess/tree/master" title="Github">}}

{{< figure align=center width=80% height=auto  src="../../chess.png" >}}

Over my life I have had a bit of fascination with the game of chess. Reinvigorated, over the period of COVID, when there was not a whole lot to do aside from learn more about the game. But looking at the engines, I was intrigued with how chess could be mastered and possibly be solved by a computational algorithm.

When looking to do this for myself I decided to use Python mostly for it's ease of use as it was the main language I had learnt up to the stage I started coding the application. As well as this, there is a hope that this engine could easily be integrated with machine learning methods in the future as there is a number of libraries for Python that would make this integration very easy.

In order to display the chess games a GUI is developed using Pygame and elements used to control the game came from the Pygame library.

## Lessons in First Implementations

First off, to see how this project went and how my thought process and skills improved we should take a look at how the first ideas in the project panned out.

During the game loop, which is the animation loop in Python, we perform functionality in a turn based system that calculates all valid moves on the board for the current board so that when the user selects two squares via a click then the check for a valid move is quick.

The main portions for this logic occurs in the function `self.board.init_valid()` which introduces a lot of inefficiencies in the code that means that the code is not particularly helpful for use in a chess engine where this function would be called many times within a search tree searching for the best move.

{{< highlight python >}}

def init_valid(self,color, basic=False):
"""
Update valid moves for all pieces of color

        :param color: int
            represents color of piece to validate
        :param basic: Bool
            whether to check if a piece is in check
        :return:
        """
        for i in self.pieces:
            s = i.update_valid_moves(self.pieces)
        if not basic:
            self.validate_check(color)

{{< /highlight >}}

First off, we find all valid moves for pieces without checking whether the opposing king is in check. The piece move checking is fairly trivial:

<!-- prettier-ignore-start -->
{{< highlight python >}}

def update_valid_moves(self, board):
        """
        Update valid moves
        :param board: List[Piece]
            list of pieces on current board
        :return:
        """
        self.valid_moves = []
        for i in self.moves:
            copy = self.pos[:]
            valid = True
            once = True
            while i[0] + copy[0] >= 0 and i[0] + copy[0] < 8 and i[1] + copy[1] >= 0 and i[1] + copy[1] < 8 and valid:
                if valid and not once:
                    valid = False
                for j in board:
                    if [i[0] + copy[0], i[1] + copy[1]] == j.get_pos() and once:
                        if j.get_color() == self.get_color():
                            valid = False
                        else:
                            valid = True
                            once = False
                if valid:
                    self.valid_moves.append([copy[0] + i[0], copy[1] + i[1]])
                copy = [copy[0] + i[0], copy[1] + i[1]]

{{< /highlight >}}
<!-- prettier-ignore-end -->

This is run for each instance of the Piece class and iterates for it's unit value of it's room (e.g. for bishop [[1,1], [-1,-1], [1,-1], [-1,1]]) and then iterate from that point onwords in those directions until encountering another piece. It is important to note that I have left out implemenetations for knight, pawn and king for brevity but it follows a similar pattern. However, you may notice that we pass a reference to `board` a list of pieces and as such there is no seperation of concerns as this method of `Piece` makes it unclear where the responsibility lies for piece generation.

The errors also go further on from a conceptual level to more efficiency based errors which are particularily evident in the code for pruning moves that result in checks.

<!-- prettier-ignore-start -->

{{< highlight python >}}
def validate_check(self,color):
        """
        Remove moves that result in current coloured King going into check

        :param color: int
            represents color of piece to validate
        :return:
        """
        copy_board = deepcopy(self)
        pieces = copy_board.pieces[:]
        for piece in pieces:
            if piece.color[0] == color:
                valid_moves = deepcopy(piece.valid_moves)
                # 4,5,6
                # 4,3,2
                if isinstance(piece, King):
                    bi_castling = [list(range(piece.pos[0],x[3][0], (x[0] - piece.pos[0])//abs(x[0] - piece.pos[0]))) for x in valid_moves if len(x) > 2]
                    for idx,direction in enumerate(bi_castling):
                        for position in direction:
                            original_pos = deepcopy(piece.pos)
                            copy_board.make_move_pos(original_pos, [position, original_pos[1]])
                            copy_board.init_valid(color, True)
                            future_check = copy_board.get_set_check(color)
                            copy_board.make_move_pos([position, original_pos[1]], original_pos)
                            copy_board.init_valid(color, True)
                            if future_check:
                                for i in self.pieces:
                                    if i.__class__.__name__ == piece.__class__.__name__ and i.color == piece.color and i.pos == piece.pos:
                                        castle_move = [x for x in valid_moves if len(x) > 2]
                                        move = castle_move[idx]
                                        for x in i.valid_moves:
                                            if [x[0],x[1]] == move[:2]:
                                                i.valid_moves.remove(move)


                for move in valid_moves:
                    original_pos = deepcopy(piece.pos)
                    moved, removed_piece = copy_board.move_piece(piece.pos, move)
                    if len(move) > 2:
                        if move[2] == "CASTLING":
                            status = move[2]
                    copy_board.init_valid(color, True)
                    future_check = copy_board.get_set_check(color)
                    copy_board.reverse_move(move, original_pos, removed_piece)
                    copy_board.init_valid(color, True)
                    if future_check:
                        for i in self.pieces:
                            if i.__class__.__name__ == piece.__class__.__name__ and i.color == piece.color and i.pos == piece.pos:
                                if (move in i.valid_moves):
                                    i.valid_moves.remove(move)

{{< /highlight >}}
<!-- prettier-ignore-end -->

There is a lot to impact here that make take a bit of time, particularily to a lack of clear comments but generally, we create a deepcopy of the board in order to make each potential move, then for each move we generate valid moves to check whether the king is in check. This is not an efficient way to perform this functionality. First of all, it is unecessary to deepcopy our board, an expensive operation, as we are simply making and then reversing moves. Next of all, we generate all valid moves in order to determine whether a piece is attacking the king (and so is in check). However this is not necessary as checks can only occur from pieces in line with the king, or at specific positions that can only be reached by specific pieces and so a lot of computation can be pruned by trying to reverse the focus from generating all moves, to generating moves for the King specifically.

I will make a lot of this clearer when we move onto the current implementation.

What will stay on though, will be the general idea of seperating game logic, state logic (logic of game starting) and UI. Specifically there are menus present which allow the user to pick the type of game (vs Computer, Player vs Player) and also their colour. This is an example of good logic seperation that makes the code somewhat easier to understand.

## Moving onto Greener Pastures

With evidence of ill-thought-out design it was time to move onto a new implementation that tries to be essentially less of a naive approach. Beginning with the `Game.py` entry file, there is a much better seperation of logic as well as explicit state management via enums that allow for clear logic. Once again, the crux of the content can be found in our `Board.py` and it's initialisation, `Board().initialise_boards()` function.

### Bitboard Magic

Keeping with the trends in chess engine development, rather than expressing the board as a 8x8 2d array with each square holding a `Piece` class, we will be making use of bitboards.

Bitboards, are useful and make sense for chess engines. This is due to the representation of the board being in a far more efficient manner.

Think about this first, rather than iterating through the rows and then columns of a 8x8 2d array we could iterate through one singular array of length 64 and index using an expression: `PieceAtRowCol(r,c) = Board[r*8 + c]`. However, how these two representations are very similar, just being two areas of contiguous memory. So, why don't we use some other datatype that is capable of storing 64 elements? How about, a 64 bit number. So that each bit set within a number corresponds to a piece? However the represention power of this method is small (we can only store one bit for each cell) so instead we can store multiple bitboards, each corresponding to a certain value.

<!-- prettier-ignore-start -->

{{< highlight python >}}
def initialise_boards(self):
        self.piece_bb[Color.WHITE][Piece.PAWN] = np.uint64(0x000000000000FF00)
        self.piece_bb[Color.WHITE][Piece.KNIGHT] = np.uint64(0x0000000000000042)
        self.piece_bb[Color.WHITE][Piece.BISHOP] = np.uint64(0x0000000000000024)
        self.piece_bb[Color.WHITE][Piece.ROOK] = np.uint64(0x0000000000000081)
        self.piece_bb[Color.WHITE][Piece.QUEEN] = np.uint64(0x0000000000000008)
        self.piece_bb[Color.WHITE][Piece.KING] = np.uint64(0x0000000000000010)

        self.piece_bb[Color.BLACK][Piece.PAWN] = np.uint64(0x00FF000000000000)
        self.piece_bb[Color.BLACK][Piece.KNIGHT] = np.uint64(0x4200000000000000)
        self.piece_bb[Color.BLACK][Piece.BISHOP] = np.uint64(0x2400000000000000)
        self.piece_bb[Color.BLACK][Piece.ROOK] = np.uint64(0x8100000000000000)
        self.piece_bb[Color.BLACK][Piece.QUEEN] = np.uint64(0x0800000000000000)
        self.piece_bb[Color.BLACK][Piece.KING] = np.uint64(0x1000000000000000)

        for p in Piece:
            for c in Color:
                self.color_occ[c] |= self.piece_bb[c][p]

        self.occ = self.color_occ[Color.WHITE] | self.color_occ[Color.BLACK]
        self.format_board.update_board(self.piece_bb, self.king_in_check)
        self.legal_moves = self.find_moves()
        self.format_board.update_legal_moves(self.legal_moves)
{{< /highlight >}}
<!-- prettier-ignore-end -->

In this code for example, bitboards for each piece are stored as a matrix so that we can access each piece bitboard by that pieces colour and piece type. Other bitboards are also in use such as the locations of all pieces on the board, and then the locations of pieces in a colour specific manner.

Don't worry though, the trusty array style of piece representation is still present, specifically in the `FormattedBoard` class which takes in the bitboards and outputs something that can be consumed by our GUI classes present in our first implementation. However, in order to extract piece locations we loop through colours then pieces to get each bitboard. Then we return the indexes of each bit that has been set in the 64 bit number. Essentially this just boils down into some math that allows us to repeatedly remove the least significant bit and get the index of the LSB, until our bitboard is empty (I may write a more in-depth blog about this step). All in all we return a representation of the board that can be used to display our GUI.

Back to logic now and finding our moves in which we introduce a new class `MoveGenerator` which also uses `MoveGenerationTable`. Our approach broadly follows our first implementation where we generate, what we now call, pseudo legal moves (moves that are legal without considering checks) and then we check for checks. First of all, let's dive deeper into the `MoveGenerationTable` in order to understand the power behind bitboards.

### Tables?

If we think about the problem, we can begin to see that the moves that each piece can make at a certain position is always constant. It is only the presence of either, other pieces or checks, that result in these moves being removed as valid moves. So rather than computing these moves in the main game loop, we can simply lookup these moves in a table that has been precomputed. The general structure of a table for a piece is a 64 long list of 64 bit numbers, each number the possible moves that the piece can make from it's index in the list.

Let's take a look at Knights!

<!-- prettier-ignore-start -->

{{< highlight python >}}
def generate_knight_moves(self):
        bb = np.zeros(64, dtype=np.uint64)
        knight_pos = np.uint64(1)

        for i in range(64):
            knight_clip_file_a_b = knight_pos & ~self.clear_files[0] & ~self.clear_files[1]
            knight_clip_file_a = knight_pos & ~self.clear_files[0]
            knight_clip_file_g_h = knight_pos & ~self.clear_files[6] & ~self.clear_files[7]
            knight_clip_file_h = knight_pos & ~self.clear_files[7]

            wwn = knight_clip_file_a_b << np.uint8(6)
            wnn = knight_clip_file_a << np.uint8(15)
            een = knight_clip_file_g_h << np.uint8(10)
            enn = knight_clip_file_h << np.uint8(17)
            ees = knight_clip_file_g_h >> np.uint8(6)
            ess = knight_clip_file_h >> np.uint8(15)
            wws = knight_clip_file_a_b >> np.uint8(10)
            wss = knight_clip_file_a >> np.uint8(17)

            bb[i] = wwn | wnn | een | enn | wws | wss | ees | ess
            knight_pos = knight_pos << np.uint8(1)
        return bb
        
{{< /highlight >}}
<!-- prettier-ignore-end -->

`bb` is the list which contains moves from each position. So we check each position in that list. It is important to note that this will not error if we go beyond a certain row or column (only if going before 0 or beyond 2^64) and so we must be rigourous in checking. There are eight possible moves a knight can make.

{{< figure align=center  width=50% height=auto  src="../../knight.png" >}}

As such, we require clipping when the knight will overflow the row, so for example, we should remove knights on row a when the knight is making moves westwards. As such, we should do a bitwise AND between our knight bitboard and a bitboard that contains 1s from index 8 onwards. So for example if our knight is on row A we would have:

<!-- prettier-ignore-start -->
{{< highlight python >}}
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |  X  | 0 1 1 1 1 1 1 1 |  =  | 0 0 0 0 0 0 0 0 |
            | 1 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
            | 0 0 0 0 0 0 0 0 |     | 0 1 1 1 1 1 1 1 |     | 0 0 0 0 0 0 0 0 |
{{< /highlight >}}
<!-- prettier-ignore-end -->

In the code this is our `clear_files` array which just stores these bitboards that mask certain files, and the same is done for ranks. As such this allows the removes knight positions when performing certain moves. Then in order to move the knight position, our clipped knight position is bit shifted to move the set bit into a proper position:

<!-- prettier-ignore-start -->

{{< highlight python>}}
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |   << 6  =  | 0 1 0 0 0 0 0 0 |
                | 0 0 0 1 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |

                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |  >> 15  =  | 0 0 0 0 0 0 0 0 |
                | 0 0 0 1 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 1 0 0 0 |
                | 0 0 0 0 0 0 0 0 |            | 0 0 0 0 0 0 0 0 |
{{< /highlight >}}
<!-- prettier-ignore-end -->

These bitshifts result in new positions for our knight that corresponds to moves. Each move direction is stored in an individual bitboard which are then ORd together to form a final bitboard containing all knight moves for a certain position.

This idea of masking for files and ranks allows us to not only perform clipping for out-of-bounds move but to also verify pawn ranks.

Generally a similar style is applied for knights, pawns and kings as they can be classed as non-sliding pieces (i.e their movements are fixed to a certain number of squares by jumping). For pawns, special considerations should be placed onto the ranks where they can shift forwards 2 squares and also the direction of that shift, as the colour is the decider for this decision.

Moving on, bishops, rook and queens follow a different logic that means we don't fully determine their moveset. The nonsliding pieces moves can be pruned simply by performing AND with free squares and as such is simple. However for sliding pieces, we will have to remove moves that come after a "blocker" and allow the moves in between the mover and the blocker.

#### Sliding Moves

The general idea is that, we can reduce the problem down to a singular rank. Each sliding piece (aside from the queen but we will get back to that) has two axes of movements. So for rooks this is along the files & ranks and for the bishop this is along the diagonal and anti-diagonal. Each of these directions of movements can be transformed, ideally into a one dimension space, as any other square on the board is irrelevant that is not a part of the possible movement direction. As such there are a possible 256 configurations of occupiers on one rank. Each cell can either be filled or not filled by an attacker. Then our piece can be placed on 8 possible squares.

As such, as part of our precomputation these the bits which can be attacked are calculated for each piece position on one rank, and then all the possible configurations of attackers. This can then be used during real-time move generation to lookup our possible moves for a certain occupation and piece location.

During move generation for sliding pieces, each direction of movement is mapped onto the first rank through a series of bit operations (which I may dive deeper into in another post). As for the queen, this is essentially just the combination of bishop and rook movements and so we can OR their movesets.

Thus we have effectively now produced legal moves for all of our pieces mostly by precomputation. This is the power of bitboards. We can use masks and bitwise operations in order to efficiently compute new positions simply that would other require iterators and other list based functions that cause too much computation.

### Our King is Under Attack!!?

But what about checks? So for this portion we have to iterate through our moves. It is simple if the king is not the piece moving:

<!-- prettier-ignore-start -->

{{< highlight python >}}
def king_is_attacked_without_move(self,board):
        king_bb = board.get_piece_bb(Piece.KING)
        king_pos = get_occupied_squares(king_bb)

        opp_color = ~board.color

        if len(king_pos) < 1:
            return True

        king_pos = king_pos[0]

        opp_pawns = board.get_piece_bb(Piece.PAWN, opp_color)
        opp_knights = board.get_piece_bb(Piece.KNIGHT, opp_color)
        opp_bishops = board.get_piece_bb(Piece.BISHOP, opp_color)
        opp_rooks = board.get_piece_bb(Piece.ROOK, opp_color)
        opp_queens = board.get_piece_bb(Piece.QUEEN, opp_color)

        pawn_check = (self.tables.pawn_moves[board.color][PawnMoveType.ATTACK][king_pos] & opp_pawns) != EMPTY_BB
        knight_check = (self.tables.knight_moves[king_pos] & opp_knights) != EMPTY_BB
        bishop_check = (self.get_bishop_moves(king_pos, board) & (opp_bishops | opp_queens)) != EMPTY_BB
        rook_check = (self.get_rook_moves(king_pos, board) & (opp_rooks | opp_queens)) != EMPTY_BB

        return pawn_check | knight_check | bishop_check | rook_check
{{< /highlight >}}
<!-- prettier-ignore-end -->

We simply behave as if the king is an attacking piece (such as a pawn, knight etc.) and check if the attacking piece's moves from the king position matches with a matching opponent's piece. If the king is being moved then we have to first of all apply the move and then perform the same action as eariler to check if the king is moving into check.

### Castling

Now castling just involved, storing castling rights for each colour and then performing the castling if the rights are held and the resulting move is invalid due to checks. Rights are granted when the spaces in between king and rook are freed and removed if either pieces move. Checking for castling just means moves the king onto squares in between castling and then checking for checks. All in all, castling is quite an involved process that results in possibly some inefficiencies and where I would like to improve in the future.

### En-Passent

En-Passent is triggered by masks that indiciate the pieces vulnerable to the attack and so a move can be performed if the piece next to a pawn is vulnerable to the attack.

### Promotions

Promotions occur when a move is actually being applied on the board and as such maybe the method `Board().applyMove()` should be introduced:

<!-- prettier-ignore-start -->

{{< highlight python >}}
def make_move(self, move, inplace=True):
        if not inplace:
            newBoard = Board()
            newBoard.pieces = np.copy(self.piece_bb)
            newBoard.color_occ = np.copy(self.color_occ)
            newBoard.occ = np.copy(self.occ)
            newBoard.color = self.color
            newBoard.legal_moves = np.copy(self.legal_moves)

        else:
            newBoard = self

        move_exists_in_legal_moves = any(
            map(lambda x: x.index_from == move.index_from and x.index_to == move.index_to, newBoard.legal_moves))

        if not move_exists_in_legal_moves:
            return newBoard

        piece = newBoard.get_piece_on(move.index_from)
        try:
            newBoard.en_passant_mask[~newBoard.color] = EMPTY_BB
        except:
            pass

        if piece == piece.KING:
            if abs(move.index_from - move.index_to) == 2:
                # Castling
                move = list(filter(lambda x: x.index_from == move.index_from and x.index_to == move.index_to,
                            newBoard.legal_moves))[0]
                newBoard.clear_square(move.rook_move.index_from)
                newBoard.set_square(move.rook_move.index_to, piece.ROOK)
                newBoard.has_moved = ~np.uint64(to_bitboard(move.rook_move.index_from)) & newBoard.has_moved
                newBoard.is_castled[newBoard.color] = True
        elif piece == piece.PAWN:
            white_promote = to_bitboard(move.index_from) & newBoard.move_generator.tables.clear_ranks[
                Rank.SEVEN] != EMPTY_BB
            black_promote = to_bitboard(move.index_from) & newBoard.move_generator.tables.clear_ranks[Rank.TWO] != EMPTY_BB
            if abs(move.index_from - move.index_to) == 16:
                newBoard.en_passant_mask[newBoard.color] = to_bitboard(move.index_to)
            elif move.index_from // 8 == 4 or move.index_from // 8 == 5:
                move = list(filter(lambda x: x.index_from == move.index_from and x.index_to == move.index_to,
                                   newBoard.legal_moves))[0]
                if move.en_passant:
                    newBoard.clear_square(move.index_to - 8 if newBoard.color == Color.WHITE else move.index_to + 8, ~newBoard.color)
            elif (newBoard.color == Color.WHITE and white_promote) or (newBoard.color != Color.WHITE and black_promote):
                piece = Piece.QUEEN


        newBoard.has_moved = ~np.uint64(to_bitboard(move.index_from)) & newBoard.has_moved

        newBoard.clear_square(move.index_from)
        newBoard.clear_square(move.index_to, color=~newBoard.color)
        newBoard.set_square(move.index_to, piece)

        newBoard.color = ~newBoard.color

        piece_bb = np.copy(newBoard.piece_bb)
        has_castled = copy.copy(newBoard.is_castled)
        color = copy.copy(newBoard.color)

        newBoard.king_in_check[newBoard.color] = newBoard.move_generator.king_is_attacked(newBoard)
        newBoard.set_board(piece_bb, has_castled, color)
        newBoard.king_in_check[~newBoard.color] = False
        newBoard.legal_moves = newBoard.find_moves()
        newBoard.game_state = newBoard.is_end_of_game()

        newBoard.history.append(copy.deepcopy(move))
        newBoard.format_board.update_board(newBoard.piece_bb, newBoard.king_in_check)
        newBoard.format_board.update_legal_moves(newBoard.legal_moves)
        newBoard.format_board.update_last_move(newBoard.history[-1])
        return newBoard
{{< /highlight >}}
<!-- prettier-ignore-end -->

First off, we allow for the copy of boards but by default this should not be done. A move is received as a parameter and is checked against our valid moves. Also at each turn we reset the current colours en-passant mask. Special logic is performed for castling pieces and promotions as they require multiple piece changes. Castling moves both the rook and queen and promotions needs a piece to dissapear and a new piece to be allocated. Promotion is decided when a pawn is moved that is currently on the penultimate rank and automatically a queen is placed in that position. En-passent is also specially worked on by setting masks of recent vulnerable pieces (double move) and since en-passent is only performed on certain ranks, those are checked and performed by checking properties set.

Squares are cleared and set and kings are checked for checks and the GUI board is updated in this method. Although possibly a method that is too long it seems fairly clear the functions the code performs.

Check out the acknowledgements for the sources and inspiration.

## Lessons Learnt

Overall, I deeply enjoyed the challenge involved in this process. From writing hard to understand and inefficient code to writing somewhat less hard to understand and inefficient code. It is written in Python of course. But, this has been fairly enjoyable process thus far. Seeing my old code gives us all a bit of a heartattack but I think it is a worthwhile endeavour. We get to see how we have along, what we have improved one, and what problems keeps giving us trouble. In particular, this project emphasised test-driven development and trying to write code that can be tested in a modular fashion so that an overall processes can be patched together by well-tested components. Also, utilising maths and logic in this comprehensive manner has been a riveting challenging on both a conceptual and practical level however the rewards are clear. Processes are simpler, code is more efficient and easier to expand and scale. I hope I can take what I have learnt here and apply them to many areas of my coding life.

### Should I Plan?

Looking at that first code attempt made me think. Should I be planning this more? Should I try to become up with a more efficient solution? As you can probably tell when I wanted to do begin I dived head first. And even though it may have resulted in inefficiencies it was worth it. I learnt more in failing and struggling in order to learn the importance of what makes code good. Even though now I might try to plan a bit more instead of going head first, maybe I will keep diving in chest first. Maybe.

## What's Next?

Now that we have an ability to play games it is time to think about who to play against. Even though I could integrate Stockfish. I would rather win. So I will be writing about the turtle-chess AI.

## Acknowledgements

- {{< newtabref href="https://pages.cs.wisc.edu/~psilord/blog/data/chess-pages/" title="Primer on Bitboards">}}
- {{< newtabref href="https://www.chessprogramming.org/Main_Page" title="Definitive source on Bitboards">}}
- {{< newtabref href="https://github.com/cglouch/snakefish/tree/master" title="Snakefish">}}

See you next time,
