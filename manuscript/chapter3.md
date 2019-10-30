# 3. Blockchain implementation

Now that we have equipped ourselves with the ability to write computer programs, we will introduce ourselves to the components (or the data structures) of what makes a cryptocurrency.

## 3.1. `wallet.rkt`

We will start with the most basic data structure - a wallet.

It is a structure that contains a public and a private key.

So it should have this form:

```racket
(struct wallet
  (private-key public-key)
  #:prefab)
```

Together with a procedure that generates a wallet. Make wallet by generating random public and private keys.

```racket
(define (make-wallet)
  (letrec ([rsa-impl (get-pk 'rsa libcrypto-factory)]
           [privkey (generate-private-key rsa-impl '((nbits 512)))]
           [pubkey (pk-key->public-only-key privkey)])
    (wallet (bytes->hex-string
             (pk-key->datum privkey 'PrivateKeyInfo))
            (bytes->hex-string
             (pk-key->datum pubkey 'SubjectPublicKeyInfo)))))

(provide (struct-out wallet) make-wallet)
```

Let's explain the code above. Additional dependencies are

```racket
(require crypto)
(require crypto/all)
```

## 3.2. `block.rkt`

We know that a block should contain the current hash, the previous hash, data, and timestamp when it was generated:

```racket
(struct block
  (hash previous-hash data timestamp))
```

SHA is a type of a hashing algorithm that allows us to confirm that the block is really what it claims to be.

In general, blocks can contain any data, not just transactions. But we are limiting it to transactions for now

```racket
(struct block
  (hash previous-hash transaction timestamp nonce)
  #:transparent)
```

And so on

The full code, explain it. Provide code without `utils.rkt` and with `utils.rkt`

```racket
(require "utils.rkt")
(require (only-in sha sha256))
(require (only-in sha bytes->hex-string))
(require racket/serialize)

(define difficulty 2)
(define target (bytes->hex-string (make-bytes difficulty 32)))

(struct block
  (hash previous-hash transaction timestamp nonce)
  #:prefab)
```

Now we have this procedure for calculating block hash:

```racket
(define (calculate-block-hash previous-hash timestamp transaction nonce)
  (bytes->hex-string (sha256 (bytes-append
           (string->bytes/utf-8 previous-hash)
           (string->bytes/utf-8 (number->string timestamp))
           (string->bytes/utf-8 (~a (serialize transaction)))
           (string->bytes/utf-8 (number->string nonce))))))
```

Now we have this procedure that determines block validity when the hash is corrcet

```racket
(define (valid-block? bl)
  (equal? (block-hash bl)
          (calculate-block-hash (block-previous-hash bl)
                                (block-timestamp bl)
                                (block-transaction bl)
                                (block-nonce bl))))
```

Now we have this procedure. A block is mined if the hash matches the target, given the difficulty.

```racket
(define (mined-block? hash)
  (equal? (subbytes (hex-string->bytes hash) 1 difficulty)
          (subbytes (hex-string->bytes target) 1 difficulty)))
```

Now we have this Hashcash implementation procedure:

```racket
(define (make-and-mine-block
         target previous-hash timestamp transaction nonce)
  (let ([hash (calculate-block-hash
               previous-hash timestamp transaction nonce)])
    (if (mined-block? hash)
        (block hash previous-hash transaction timestamp nonce)
        (make-and-mine-block
         target previous-hash timestamp transaction (+ nonce 1)))))
```

Now we have this procedure which is a wrapper around `make-and-mine-block`.

```racket
(define (mine-block transaction previous-hash)
  (make-and-mine-block
   target previous-hash (current-milliseconds) transaction 1))
```

Finally

```racket
(provide (struct-out block) mine-block valid-block? mined-block?)
```

## 3.3. Transactions

Public / private key explanation

Signing and verifying signatures

### 3.3.1. `transaction-io.rkt`

Explain this code

```racket
(require (only-in sha sha256))
(require (only-in sha bytes->hex-string))
(require racket/serialize)

(struct transaction-io
  (hash value owner timestamp)
  #:prefab)
```

Now we have this procedure for calculating the hash of a `transaction-io` object

```racket
(define (calculate-transaction-io-hash value owner timestamp)
  (bytes->hex-string (sha256 (bytes-append
           (string->bytes/utf-8 (number->string value))
           (string->bytes/utf-8 (~a (serialize owner)))
           (string->bytes/utf-8 (number->string timestamp))))))
```

Now we have this procedure that makes a `transaction-io` object with calculated hash:

```racket
(define (make-transaction-io value owner)
  (let ([timestamp (current-milliseconds)])
    (transaction-io
     (calculate-transaction-io-hash value owner timestamp)
     value
     owner
     timestamp)))
```

Now we have this procedure that determines `transaction-io` validity when the hash is correct:

```racket
(define (valid-transaction-io? t-in)
  (equal? (transaction-io-hash t-in)
          (calculate-transaction-io-hash
           (transaction-io-value t-in)
           (transaction-io-owner t-in)
           (transaction-io-timestamp t-in))))
```

Finally

```racket
(provide (struct-out transaction-io)
         make-transaction-io valid-transaction-io?)
```

### 3.3.2. `transaction.rkt`

Explain this code.

```racket
(require "transaction-io.rkt")
(require "utils.rkt")
(require "wallet.rkt")
(require crypto)
(require crypto/all)
(require racket/serialize)

(struct transaction
  (signature from to value inputs outputs)
  #:prefab)
```

In addition, we need to use all crypto factories for converting the key between `hex<->pk-key`:

```racket
(use-all-factories!)
```

Now we have this procedure that returns digested signature of a transaction data:

```racket
(define (sign-transaction from to value)
  (let ([privkey (wallet-private-key from)]
        [pubkey (wallet-public-key from)])
    (bytes->hex-string
     (digest/sign
      (datum->pk-key (hex-string->bytes privkey) 'PrivateKeyInfo)
      'sha1
      (bytes-append
       (string->bytes/utf-8 (~a (serialize from)))
       (string->bytes/utf-8 (~a (serialize to)))
       (string->bytes/utf-8 (number->string value)))))))
```

Now we have this procedure that makes an empty, unprocessed and unsigned transaction:

```racket
(define (make-transaction from to value inputs)
  (transaction
   ""
   from
   to
   value
   inputs
   '()))
```

Now we have this procedure for processing transactions. It sums all the inputs with `inputs-sum`, calculates the `leftover` and then generates new outputs to be used in the new signed and processed transaction `new-outputs`:

```racket
(define (process-transaction t)
  (letrec
      ([inputs (transaction-inputs t)]
       [outputs (transaction-outputs t)]
       [value (transaction-value t)]
       [inputs-sum
        (foldr + 0 (map (lambda (i) (transaction-io-value i)) inputs))]
       [leftover (- inputs-sum value)]
       [new-outputs
        (list
         (make-transaction-io value (transaction-to t))
         (make-transaction-io leftover (transaction-from t)))])
    (transaction
     (sign-transaction (transaction-from t)
                       (transaction-to t)
                       (transaction-value t))
     (transaction-from t)
     (transaction-to t)
     value
     inputs
     (remove-duplicates (append new-outputs outputs)))))
```

Now we have this procedure that checks the signature validity of a transaction:

```racket
(define (valid-transaction-signature? t)
  (let ([pubkey (wallet-public-key (transaction-from t))])
    (digest/verify
     (datum->pk-key (hex-string->bytes pubkey) 'SubjectPublicKeyInfo)
     'sha1
     (bytes-append
      (string->bytes/utf-8 (~a (serialize (transaction-from t))))
      (string->bytes/utf-8 (~a (serialize (transaction-to t))))
      (string->bytes/utf-8 (number->string (transaction-value t))))
     (hex-string->bytes (transaction-signature t)))))
```

Now we have this procedure that determines transaction validity when:
1. Its signature is valid `valid-transaction-signature?`
1. All outputs are valid `valid-transaction-io?`
1. The sum of the inputs is gte the sum of the outputs `>= sum-inputs sum-outputs`

```racket
(define (valid-transaction? t)
  (let ([sum-inputs
         (foldr + 0 (map (lambda (t) (transaction-io-value t))
                         (transaction-inputs t)))]
        [sum-outputs
         (foldr + 0 (map (lambda (t) (transaction-io-value t))
                         (transaction-outputs t)))])
    (and
     (valid-transaction-signature? t)
     (true-for-all? valid-transaction-io? (transaction-outputs t))
     (>= sum-inputs sum-outputs))))
```

Finally

```racket
(provide (all-from-out "transaction-io.rkt")
         (struct-out transaction)
         make-transaction process-transaction valid-transaction?)
```

## 3.4. `blockchain.rkt`

UTXO Why? Performance reasons.

```racket
(require "block.rkt")
(require "transaction.rkt")
(require "utils.rkt")
(require "wallet.rkt")

(struct blockchain
  (blocks utxo)
  #:prefab)
```

Now we have this procedure for initialization of the blockchain:

```racket
(define (init-blockchain t seed-hash utxo)
  (blockchain (cons (mine-block (process-transaction t) seed-hash) '())
              utxo))
```

Now we have this procedure. We start with 50 coins initially, and halve them on every 210000 blocks.

```racket
(define (mining-reward-factor blocks)
  (/ 50 (expt 2 (floor (/ (length blocks) 210000)))))
```

Now we have this procedure that adds a transaction to blockchain by processing the unspent transaction outputs:

```racket
(define (add-transaction-to-blockchain b t)
  (letrec ([hashed-blockchain
            (mine-block t (block-hash (car (blockchain-blocks b))))]
           [processed-inputs (transaction-inputs t)]
           [processed-outputs (transaction-outputs t)]
           [utxo (set-union processed-outputs
                            (set-subtract (blockchain-utxo b)
                                          processed-inputs))]
           [new-blocks (cons hashed-blockchain (blockchain-blocks b))]
           [utxo-rewarded (cons
                           (make-transaction-io
                            (mining-reward-factor new-blocks)
                            (transaction-from t))
                           utxo)])
    (blockchain
     new-blocks
     utxo-rewarded)))
```

Now we have this procedure that sends money from one wallet to another by initiating transaction, and then adding it to the blockchain for processing:

```racket
(define (send-money-blockchain b from to value)
  (letrec ([my-ts
            (filter (lambda (t) (equal? from (transaction-io-owner t)))
                    (blockchain-utxo b))]
           [t (make-transaction from to value my-ts)])
    (if (transaction? t)
        (let ([processed-transaction (process-transaction t)])
          (if (and (>= (balance-wallet-blockchain b from) value)
                   (valid-transaction? processed-transaction))
              (add-transaction-to-blockchain b processed-transaction)
              b))
        (add-transaction-to-blockchain b '()))))
```

Now we have this procedure that determines the balance of a wallet - the sum of all unspent transactions for the matching owner:

```racket
(define (balance-wallet-blockchain b w)
  (letrec ([utxo (blockchain-utxo b)]
           [my-ts (filter
                   (lambda (t) (equal? w (transaction-io-owner t)))
                   utxo)])
    (foldr + 0 (map (lambda (t) (transaction-io-value t)) my-ts))))
```

Now we have this procedure that determines blockchain validity:

1. All blocks are valid `valid-block?`
1. Previous hashes are matching `equal?` check
1. All transactions are valid `valid-transaction?`
1. All blocks are mined `mined-block?`

```racket
(define (valid-blockchain? b)
  (let ([blocks (blockchain-blocks b)])
    (and
     (true-for-all? valid-block? blocks)
     (equal? (drop-right (map block-previous-hash blocks) 1)
             (cdr (map block-hash blocks)))
     (true-for-all?
      valid-transaction? (map
                          (lambda (block) (block-transaction block))
                          blocks))
     (true-for-all?
      mined-block? (map block-hash blocks)))))
```

Finally

```racket
(provide (all-from-out "block.rkt")
         (all-from-out "transaction.rkt")
         (all-from-out "wallet.rkt")
         (struct-out blockchain)
         init-blockchain send-money-blockchain
         balance-wallet-blockchain valid-blockchain?)
```

## 3.6. `utils.rkt`

```racket
(require racket/serialize)

(define ASCII-ZERO (char->integer #\0))
```

Now we have this procedure that converts a hexadecimal character matching [0-9A-Fa-f] to a number from 0 to 15:

```racket
(define (hex-char->number c)
  (if (char-numeric? c)
      (- (char->integer c) ASCII-ZERO)
      (match c
        [(or #\a #\A) 10]
        [(or #\b #\B) 11]
        [(or #\c #\C) 12]
        [(or #\d #\D) 13]
        [(or #\e #\E) 14]
        [(or #\f #\F) 15]
        [_ (error 'hex-char->number "invalid hex char: ~a\n" c)])))
```

Now we have this procedure that converts a hex string to bytes:

```racket
(define (hex-string->bytes str)
  (list->bytes (hex-string->bytelist str)))

(define (hex-string->bytelist str)
  (with-input-from-string
      str
    (thunk
     (let loop ()
       (define c1 (read-char))
       (define c2 (read-char))
       (cond [(eof-object? c1) null]
             [(eof-object? c2) (list (hex-char->number c1))]
             [else (cons (+ (* (hex-char->number c1) 16)
                            (hex-char->number c2))
                         (loop))])))))
```

Now we have this procedure that returns true if the predicate satisfies all members of the list:

```racket
(define (true-for-all? pred list)
  (cond
    [(empty? list) #t]
    [(pred (first list)) (true-for-all? pred (rest list))]
    [else #f]))
```

Now we have this procedure for exporting a struct to a file:

```racket
(define (struct->file object file)
  (let ([out (open-output-file file #:exists 'replace)])
    (write (serialize object) out)
    (close-output-port out)))
```

Now we have this procedure for importing struct contents from a file:

```racket
(define (file->struct file)
  (letrec ([in (open-input-file file)]
           [result (read in)])
    (close-input-port in)
    (deserialize result)))
```

Finally

```racket
(provide hex-string->bytes true-for-all? struct->file file->struct)
```

## 3.7. Integrating components

### 3.7.1. `main-helper.rkt`

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

### 3.7.2. `main.rkt`

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

## Summary

Components are orthogonal. This means that every component is independent of one another, that is, wallet's implementation does not call procedures in block for example, and that a block can be used independently of wallet.

But when combined we get a cryptocurrency system.

What are the gains?
