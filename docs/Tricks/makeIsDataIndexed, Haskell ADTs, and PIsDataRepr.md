# The isomorphism between `makeIsDataIndexed`, Haskell ADTs, and `PIsDataRepr`

When [implementing `PIsDataRepr`](./../Typeclasses/PIsDataRepr%20and%20PDataFields.md#implementing-pisdatarepr-and-friends) for a Plutarch type, if the Plutarch type also has a [Haskell synonym](./../Concepts/Haskell%20Synonym.md) (e.g. `ScriptContext` is the Haskell synonym to `PScriptContext`) that uses [`makeIsDataIndexed`](https://playground.plutus.iohkdev.io/doc/haddock/plutus-tx/html/PlutusTx.html#v:makeIsDataIndexed) - you must make sure the constructor ordering is correct.

In particular, with `makeIsDataIndexed`, you can assign _indices_ to your Haskell ADT's constructors. This determines how the ADT will be represented in Plutus Core. It's important to ensure that the corresponding Plutarch type _knows_ about these indices so it can decode the ADT correctly - in case you passed it into Plutarch code, through Haskell.

For example, consider `Maybe`. Plutus assigns these indices to its constructors:

```hs
makeIsDataIndexed ''Maybe [('Just, 0), ('Nothing, 1)]
```

0 to `Just`, 1 to `Nothing`. So the corresponding Plutarch type, `PMaybeData` is defined as:

```hs
data PMaybeData a (s :: S)
  = PDJust (Term s (PDataRecord '["_0" ':= a]))
  | PDNothing (Term s (PDataRecord '[]))
```

It'd be a very subtle mistake to instead define it as:

```hs
data PMaybeData a (s :: S)
  = PDNothing (Term s (PDataRecord '[]))
  | PDJust (Term s (PDataRecord '["_0" ':= a]))
```

The constructor ordering is wrong!

It's not just constructor ordering that matters - field ordering does too! Though this is self explanatory. Notice how `PTxInfo` shares the exact same field ordering as its Haskell synonym - `TxInfo`.

```hs
newtype PTxInfo (s :: S)
  = PTxInfo
      ( Term
          s
          ( PDataRecord
              '[ "inputs" ':= PBuiltinList (PAsData PTxInInfo)
               , "outputs" ':= PBuiltinList (PAsData PTxOut)
               , "fee" ':= PValue
               , "mint" ':= PValue
               , "dcert" ':= PBuiltinList (PAsData PDCert)
               , "wdrl" ':= PBuiltinList (PAsData (PTuple PStakingCredential PInteger))
               , "validRange" ':= PPOSIXTimeRange
               , "signatories" ':= PBuiltinList (PAsData PPubKeyHash)
               , "data" ':= PBuiltinList (PAsData (PTuple PDatumHash PDatum))
               , "id" ':= PTxId
               ]
          )
      )
```

```hs
data TxInfo = TxInfo
  { txInfoInputs      :: [TxInInfo]
  , txInfoOutputs     :: [TxOut]
  , txInfoFee         :: Value
  , txInfoMint        :: Value
  , txInfoDCert       :: [DCert]
  , txInfoWdrl        :: [(StakingCredential, Integer)]
  , txInfoValidRange  :: POSIXTimeRange
  , txInfoSignatories :: [PubKeyHash]
  , txInfoData        :: [(DatumHash, Datum)]
  , txInfoId          :: TxId
  }
```

The _field names_ don't matter though. They are merely labels that don't exist at runtime.
