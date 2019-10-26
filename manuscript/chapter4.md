# 4. Component implementation

Put code from https://github.com/bor0/scheme-coin

## 4.1. Wallet

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

## 4.2. Block

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

## 4.3. Transactions

Public / private key explanation

Signing and verifying signatures

## 4.3.1. Transaction IO

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

## 4.3.2. Transaction

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

## 4.4. Blockchain

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

## 4.5. Peer-to-peer

```racket
(require "blockchain.rkt")
(require "block.rkt")
(require racket/serialize)
```

The `peer-info` structure contains an ip and a port:

```racket
(struct peer-info
  (ip port)
  #:prefab)
```

`peer-info-io` additionally contains IO ports:

```racket
(struct peer-info-io
  (pi input-port output-port)
  #:prefab)
```

`peer-context-data` contains all information needed for a single peer:

```racket
(struct peer-context-data
  (name
   port
   [valid-peers #:mutable]
   [connected-peers #:mutable]
   [blockchain #:mutable])
  #:prefab)
```

The list of valid peers will be updated depending on info retrieved from connected peers. The list of connected peers will be a (not necessarily strict) subset of `valid-peers`. Blockchain will be updated from data with other peers.

Now we have this procedure for getting the sum of nonces of a blockchain. Highest one has most effort and will win to get updated throughout the peers.

```racket
(define (get-blockchain-effort b)
  (foldl + 0 (map block-nonce (blockchain-blocks b))))
```

Now we have this procedure which will server as a handler for updating latest blockchain:

TODO: implement `(define (helper x) (deserialize (read (open-input-string (string-replace-line x "")))))`

```racket
(define (maybe-update-blockchain peer-context line)
  (let ([current-blockchain
         (deserialize
          (read
           (open-input-string
            (string-replace line
                            #rx"(latest-blockchain:|[\r\n]+)" ""))))]
        [latest-blockchain (peer-context-data-blockchain peer-context)])
    (when (and (valid-blockchain? current-blockchain)
               (> (get-blockchain-effort current-blockchain)
                  (get-blockchain-effort latest-blockchain)))
      (printf "Blockchain updated for peer ~a\n"
              (peer-context-data-name peer-context))
      (set-peer-context-data-blockchain! peer-context
                                         current-blockchain))))
```

Together with this handler for updating valid peers:

```racket
(define (maybe-update-valid-peers peer-context line)
  (let ([valid-peers
         (list->set
          (deserialize
           (read
            (open-input-string
             (string-replace line #rx"(valid-peers:|[\r\n]+)" "")))))]
        [current-valid-peers (peer-context-data-valid-peers
                              peer-context)])
    (set-peer-context-data-valid-peers!
     peer-context
     (set-union current-valid-peers valid-peers))))
```

Now we have these generic handlers for both client and server: `handler`, `launch-handler-thread` and `peers`.

`handler` is the main handler:

```racket
(define (handler peer-context in out)
  (flush-output out)
  (define line (read-line in))
  (when (string? line) ; it can be eof
    (cond [(string-prefix? line "get-valid-peers")
           (begin (display "valid-peers:" out)
                  (displayln
                   (serialize
                    (set->list
                     (peer-context-data-valid-peers peer-context)))
                   out)
                  (handler peer-context in out))]
          [(string-prefix? line "get-latest-blockchain")
           (begin (display "latest-blockchain:" out)
                  (write (serialize
                          (peer-context-data-blockchain peer-context))
                         out)
                  (handler peer-context in out))]
          [(string-prefix? line "latest-blockchain:")
           (begin (maybe-update-blockchain peer-context line)
                  (handler peer-context in out))]
          [(string-prefix? line "valid-peers:")
           (begin (maybe-update-valid-peers peer-context line)
                  (handler peer-context in out))]
          [(string-prefix? line "exit")
           (displayln "bye" out)]
          [else (handler peer-context in out)])))
```

Now we have this helper procedure to launch handler thread:

```racket
(define (launch-handler-thread handler peer-context in out cb)
  (define-values (local-ip remote-ip) (tcp-addresses in))
  (define current-peer (peer-info
                        remote-ip
                        (peer-context-data-port peer-context)))
  (define current-peer-io (peer-info-io current-peer in out))
  (thread
   (lambda ()
     (handler peer-context in out)
     (cb)
     (close-input-port in)
     (close-output-port out))))
```

Now we have this procedure that will ping all peers in attempt to sync blockchains and update list of valid peers:

```racket
(define (peers peer-context)
  (define (loop)
    (sleep 10)
    (for [(p (peer-context-data-connected-peers peer-context))]
      (let ([in (peer-info-io-input-port p)]
            [out (peer-info-io-output-port p)])
        (displayln "get-latest-blockchain" out)
        (displayln "get-valid-peers" out)
        (flush-output out)))
    (printf "Peer ~a reports ~a valid peers.\n"
            (peer-context-data-name peer-context)
            (set-count (peer-context-data-valid-peers peer-context)))
    (loop))
  (define t (thread loop))
  (lambda ()
    (kill-thread t)))
```

Now we have these two generic procedures for server: `accept-and-handle` and `serve`.

`accept-and-handle` accepts a new connection:

```racket
(define (accept-and-handle listener handler peer-context)
  (define-values (in out) (tcp-accept listener))
  (launch-handler-thread handler peer-context in out void))
```

`serve` is the main server listener:

```racket
(define (serve peer-context)
  (define main-cust (make-custodian))
  (parameterize ([current-custodian main-cust])
    (define listener
      (tcp-listen (peer-context-data-port peer-context) 5 #t))
    (define (loop)
      (accept-and-handle listener handler peer-context)
      (loop))
    (thread loop))
  (lambda ()
    (custodian-shutdown-all main-cust)))
```

Now we have this generic procedure for client, `connections-loop`, that makes sure we're connected with all known peers. TODO: break this procedure into smaller parts?

```racket
(define (connections-loop peer-context)
  (define conns-cust (make-custodian))
  (parameterize ([current-custodian conns-cust])
    (define (loop)
      (letrec ([current-connected-peers (list->set (map peer-info-io-pi (peer-context-data-connected-peers peer-context)))]
               [all-valid-peers (peer-context-data-valid-peers peer-context)]
               [potential-peers (set-subtract all-valid-peers current-connected-peers)])
        (for ([peer potential-peers])
          (thread (lambda ()
                    (with-handlers
                        ([exn:fail?
                          (lambda (x)
                            ;(printf "Cannot connect to ~a:~a\n" (peer-info-ip peer) (peer-info-port peer))
                            #t)])
                      (begin
                        ;(printf "Trying to connect to ~a:~a...\n" (peer-info-ip peer) (peer-info-port peer))
                        (define-values (in out) (tcp-connect (peer-info-ip peer) (peer-info-port peer)))
                        (printf "'~a' connected to ~a:~a!\n" (peer-context-data-name peer-context) (peer-info-ip peer) (peer-info-port peer))
                        (define current-peer-io (peer-info-io peer in out))
                        ; Add current peer to list of connected peers
                        (set-peer-context-data-connected-peers! peer-context (cons current-peer-io (peer-context-data-connected-peers peer-context)))
                        (launch-handler-thread handler
                                               peer-context
                                               in
                                               out
                                               (lambda ()
                                                 ; Remove peer from list of connected peers
                                                 (set-peer-context-data-connected-peers! peer-context
                                                                                         (set-remove
                                                                                          (peer-context-data-connected-peers peer-context)
                                                                                          current-peer-io)))))))))
        (sleep 10)
        (loop)))
    (thread loop))
  (lambda ()
    (custodian-shutdown-all conns-cust)))
```

Now we have this helper procedure for running a peer-to-peer connection.

```racket
(define (run-peer peer-context)
  (let ([stop-listener (serve peer-context)]
        [stop-peers-loop (peers peer-context)]
        [stop-connections-loop (connections-loop peer-context)])
    (lambda ()
      (begin
        (stop-connections-loop)
        (stop-peers-loop)
        (stop-listener)))))
```

Finally

```racket
(provide (struct-out peer-context-data)
         (struct-out peer-info)
         run-peer)
```

## 4.6. Utils

utils.rkt is

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

Components are orthogonal. This means that every component is independent of one another, that is, wallet's implementation does not call procedures in block for example, and that a block can be used independently of wallet.
But when combined we get a cryptocurrency system.

What are the gains?

- Immutability
- Game WoW example players (peers) implementing their own rules, etc
