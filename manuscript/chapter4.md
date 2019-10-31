# 4. Peer-to-peer implementation

## 4.1. `peer-to-peer.rkt`

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

## 4.2. Updated `main-helper.rkt`

Modify `main-helper.rkt` to also include peer-to-peer implementation:

```racket
; ...
(require "peer-to-peer.rkt")
; ...

(provide (all-from-out "blockchain.rkt")
         (all-from-out "utils.rkt")
         (all-from-out "peer-to-peer.rkt")
         format-transaction print-block print-blockchain print-wallets)
```

## 4.3. `main-p2p.rkt`

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
