# 5. Integrating components

`main-helper.rkt`:

```racket
(require "./src/blockchain.rkt")
(require "./src/utils.rkt")
(require "./src/peer-to-peer.rkt")

(require (only-in sha bytes->hex-string))

(define (format-transaction t)
  (format "...~a... sends ...~a... an amount of ~a."
          (substring (wallet-public-key (transaction-from t)) 64 80)
          (substring (wallet-public-key (transaction-to t)) 64 80)
          (transaction-value t)))

(define (print-block bl)
  (printf "Block information\n=================\nHash:\t~a\nHash_p:\t~a\nStamp:\t~a\nNonce:\t~a\nData:\t~a\n"
          (block-hash bl)
          (block-previous-hash bl)
          (block-timestamp bl)
          (block-nonce bl)
          (format-transaction (block-transaction bl))))

(define (print-blockchain b)
  (for ([block (blockchain-blocks b)])
    (print-block block)
    (newline)))

(define (print-wallets b wallet-a wallet-b)
  (printf "\nWallet A balance: ~a\nWallet B balance: ~a\n\n"
          (balance-wallet-blockchain b wallet-a)
          (balance-wallet-blockchain b wallet-b)))

(provide (all-from-out "./src/blockchain.rkt")
         (all-from-out "./src/utils.rkt")
         (all-from-out "./src/peer-to-peer.rkt")
         format-transaction print-block print-blockchain print-wallets)
```

`main-p2p.rkt`

```racket
(require "./main-helper.rkt")
```

This will convert a string of format `ip:port` to `peer-info` structure:

```racket
(define (string-to-peer-info s)
  (let ([s (string-split s ":")])
    (peer-info (car s) (string->number (cadr s)))))
```

This will create a new wallet for us to use:

```racket
(define wallet-a (make-wallet))
```

Finally, the creation of new blockchain that initializes wallets, transactions, unspect transactions and blockchain:

```racket
(define (initialize-new-blockchain)
  (begin
    (define scheme-coin-base (make-wallet))

    (printf "Making genesis transaction...\n")
    (define genesis-t
      (make-transaction scheme-coin-base wallet-a 100 '()))

    (define utxo (list
                  (make-transaction-io 100 wallet-a)))

    (printf "Mining genesis block...\n")
    (define b (init-blockchain genesis-t "1337cafe" utxo))
    b))
```

Some command line parsing (what's a command line to a newbie?)

```racket
(define args (vector->list (current-command-line-arguments)))

(when (not (= 3 (length args)))
  (begin
    (printf
     "Usage: racket main-p2p.rkt dbfile.data port ip1:port1,...\n")
    (exit)))

(define db-filename (car args))
(define port (string->number (cadr args)))
(define valid-peers
  (map string-to-peer-info (string-split (caddr args) ",")))
```

This will try to read the blockchain from a file (DB), otherwise create a new one:

```racket
(define b
  (if (file-exists? db-filename)
      (file->struct db-filename)
      (initialize-new-blockchain)))
```

Peer initialization:

```racket
(define peer-context
  (peer-context-data "Test peer" port (list->set valid-peers) '() b))
(define (get-blockchain) (peer-context-data-blockchain peer-context))

(run-peer peer-context)
```

We keep exporting the database to have up-to-date info whenever a user quits the app.

```racket
(define (export-loop)
  (begin
    (sleep 10)
    (struct->file (get-blockchain) db-filename)
    (printf "Exported blockchain to '~a'...\n" db-filename)
    (export-loop)))

(thread export-loop)
```

Here's a procedure to keep mining empty blocks, as the p2p runs in threaded mode.

```racket
(define (mine-loop)
  (let ([newer-blockchain
         ; This blockchain includes a new block
         (send-money-blockchain (get-blockchain) wallet-a wallet-a 1)])
    (set-peer-context-data-blockchain! peer-context newer-blockchain)
    (displayln "Mined a block!")
    (sleep 5)
    (mine-loop)))

(mine-loop)
```

`main.rkt`

```racket
(require "./main-helper.rkt")

(when (file-exists? "blockchain.data")
  (begin
    (printf "Found 'blockchain.data', reading...\n")
    (print-blockchain (file->struct "blockchain.data"))
    (exit)))
```

We initialize wallets:

```racket
(define scheme-coin-base (make-wallet))
(define wallet-a (make-wallet))
(define wallet-b (make-wallet))
```

We initialize transactions:

```racket
(printf "Making genesis transaction...\n")
(define genesis-t (make-transaction scheme-coin-base wallet-a 100 '()))
```

We initialize unspent transactions (store our genesis transaction):

```racket
(define utxo (list
              (make-transaction-io 100 wallet-a)))
```

Here's the blockchain initiation

```racket
(printf "Mining genesis block...\n")
(define blockchain (init-blockchain genesis-t "1337cafe" utxo))
(print-wallets blockchain wallet-a wallet-b)
```

Make a second transaction

```racket
(printf "Mining second transaction...\n")
(set! blockchain (send-money-blockchain blockchain wallet-a wallet-b 2))
(print-wallets blockchain wallet-a wallet-b)
```

Make a third transaction

```racket
(printf "Mining third transaction...\n")
(set! blockchain (send-money-blockchain blockchain wallet-b wallet-a 1))
(print-wallets blockchain wallet-a wallet-b)
```

Attempt to make a fourth transaction

```racket
(printf "Attempting to mine fourth (not-valid) transaction...\n")
(set! blockchain (send-money-blockchain blockchain wallet-b wallet-a 3))
(print-wallets blockchain wallet-a wallet-b)
```

Check validity

```racket
(printf "Blockchain is valid: ~a\n\n" (valid-blockchain? blockchain))
```

Print contents

```racket
(for ([block (blockchain-blocks blockchain)])
  (print-block block)
  (newline))
```

And export

```racket
(struct->file blockchain "blockchain.data")
(printf "Exported blockchain to 'blockchain.data'...\n")
```
