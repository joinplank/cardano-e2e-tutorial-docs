.. _onchain:

On-Chain
========

The on-chain side of the server implementation consists of validators and minting policies.
In our example, the sender locks into a script UTxO the tokens to be paid to the
receiver, containing in the datum all the necessary information. In order to check
that it is valid, a `control token` is minted in the same transaction,
so validation can be performed by running the corresponding minting policy.
For canceling or resolving it is necessary to spend the script UTxO, so the
corresonding validator performs all the necessary checks.

In the :code:`OnChain` module we find the Haskell implementation of the
validator and the minting policy as boolean functions.
In the :code:`Validator` module we define mostly boilerplate code for compiling to Plutus.

OnChain Module
--------------

Because all the functions on this module will be compiled to Plutus we must
use the *Plutus Prelude* instead of the *Haskell Prelude*. To avoid confusion about
which Prelude we are using, we add the ``NoImplicitPrelude`` pragma at the top of
the module. As a result we don't have any of the common types and functions in this
scope. Thus we must explictly import from the Prelude all we need:

.. code:: Haskell

   import PlutusTx.Prelude ( Integer, Bool
                           , ($), (&&), (||), (==)
                           , traceIfFalse
                           )

Minting Policy
~~~~~~~~~~~~~~

Starting an escrow involves minting a *control token*, thus running the validation
implemented on its minting policy. We use this
fact to ensure some good starting properties on the script UTxO we are creating.
A similar approach is proposed in this
`blogpost <https://well-typed.com/blog/2022/08/plutus-initial-conditions/>`_.
For that, we have to distinguish if we are minting or burning by checking the
information in the script context: if the amount of minted tokens is -1 we
are burning, if it is 1 we are minting. If the transaction mints a different
amount of tokens, the minting policy fails and so does the start operation.
If we are minting the control token, then we have to check that it is paid to
the escrow script and that the datum is correct:

.. code:: Haskell

   {-# INLINABLE mkControlTokenMintingPolicy #-}
   mkControlTokenMintingPolicy :: ScriptAddress -> () -> ScriptContext -> Bool
   mkControlTokenMintingPolicy addr _ ctx =
       traceIfFalse "Burning less or more than one control token" (mintedA == -1)
           ||
           (   traceIfFalse "Minting more than one control token"
                            (mintedA == 1)
            && traceIfFalse "The control token was not paid to the script address"
                            controlTokenPaid
            && traceIfFalse "Wrong information in Datum"
                            correctDatum
           )
     where
       ....

The minting policy is parameterized on the script address of the escrow UTxO we
are creating, so we know where the control token must go. We check that it is
paid to the script UTxO in ``controlTokenPaid``:

.. code:: Haskell

      controlTokenPaid :: Bool
      controlTokenPaid =
          assetClassValueOf (txOutValue escrowUtxo) (assetClass mintedCS mintedTN)
          ==
          mintedA

      escrowUtxo :: TxOut
      escrowUtxo = getSingleton $ outputsAt addr info

      mintedCS :: CurrencySymbol
      mintedTN :: TokenName
      mintedA :: Integer
      (mintedCS, mintedTN, mintedA) = getSingleton $
                                      flattenValue $ txInfoMint info
      

Here, the escrow UTxO output is obtained using ``outputsAt``, and with the
utility ``getSingleton`` we get it from the list or fail if there is more than
one output at the same address.
Also, the minting information is extracted by using ``flattenValue`` together
with ``getSingleton``, failing if it is found that more than one asset is being
minted.

To ensure that the datum is correct we need to check that the sender address
corresponds to the transaction signer's payment key, the amount of tokens to
receive is greater than zero, and the control token Asset Class is the one that
is being minted.

.. code:: Haskell

      correctDatum :: Bool
      correctDatum =
          traceIfFalse "The signer is not the sender on the escrow"
                       correctSigner
       && traceIfFalse "The asset minted does not match with the control token"
                       correctControlAssetClass
       && traceIfFalse "The receive amount of tokens to exchange is not positive"
                       correctAmount

For implementing these three checks we simply read the script UTxO datum and
compare its information with the expected one.


Validator
~~~~~~~~~

The on-chain validator, as we briefly mentioned, is parameterized on the receiver
address. This design decision allows us to have a unique script address for each
``ReceiverAddress``. Given that we are minting a control token, it would be
desired to include in the parameter its asset class for checking that it is
burned when canceling or resolving. However, we can't do this because of
a circularity problem: in the minting policy we need the script address for
ensuring that the token is paid to the corresponding UTxO. We solve
this issue by including the control token asset class in the datum, as we
showed before.

The validator will run when the script UTxO is spent, and it corresponds to `Cancel` and
`Resolve` operations, which are the only two constructors of :code:`EscrowRedeemer` type. In both
cases we have to check that the control token is burned and only one script UTxO is spent.
The latter check is important for preventing `double satisfaction attacks <https://plutus.readthedocs.io/en/latest/reference/writing-scripts/common-weaknesses/double-satisfaction.html>`_. 

.. code:: haskell

   {-# INLINABLE mkEscrowValidator #-}
   mkEscrowValidator :: ReceiverAddress
                     -> EscrowDatum
                     -> EscrowRedeemer
                     -> ScriptContext
                     -> Bool
   mkEscrowValidator raddr EscrowDatum{..} r ctx =
       case r of
           CancelEscrow  -> cancelValidator eInfo signer
           ResolveEscrow -> resolveValidator info eInfo raddr signer scriptValue
      &&
      traceIfFalse "more than one script input utxo"
                   (length sUtxos == 1)
      &&
      traceIfFalse "controlToken was not burned"
                   (eAssetClass == assetClass mintedCS mintedTN && mintedA == -1)
     where
       info :: TxInfo
       info = scriptContextTxInfo ctx

       signer :: PubKeyHash
       signer = getSingleton $ txInfoSignatories info

       scriptValue :: Value
       scriptValue = txOutValue (getSingleton sUtxos)
                   <> assetClassValue eAssetClass (-1)

       .....


We modularize the validator implementing functions for each case:
:code:`cancelValidator` and :code:`resolveValidator`. For implementing the
first one we need the Escrow Info (which is inside the datum) and the signer
(which is extracted from the Script Context). For implementing the second one
we also pass the entire Script Context info, the validator parameter (the
receiver address), and the value contained inside the script utxo that
must be paid to the receiver address (it's the whole value less the control token,
which is burned).

      
Validating a cancel operation is simple: we have to check that the escrow sender
is the one signing the transaction. 

.. code:: haskell

   {-# INLINABLE cancelValidator #-}
   cancelValidator :: EscrowInfo -> PubKeyHash -> Bool
   cancelValidator EscrowInfo{..} signer =
       traceIfFalse "cancelValidator: Wrong sender signature"
                    $ signerIsSender signer sender

The sender address is stored in the datum at start, so at canceling we check that
the information in the datum coincides with the transaction signer.

A more interesting validation is required for resolving an escrow. We check that
the signer is the receiver, and payments to sender and receiver addresses are
correct.


.. code:: haskell

  {-# INLINABLE resolveValidator #-}
  resolveValidator
      :: TxInfo
      -> EscrowInfo
      -> ReceiverAddress
      -> PubKeyHash
      -> Value
      -> Bool
  resolveValidator info ei raddr@ReceiverAddress{..} signer scriptValue =
      traceIfFalse "resolveValidator: Wrong receiver signature"
                   (signerIsReceiver signer raddr)
      &&
      traceIfFalse "resolveValidator: Wrong sender's payment"
                   (valueToSender ei `leq` senderV)
      &&
      traceIfFalse "resolveValidator: Wrong receiver's payment"
                   (scriptValue `leq` receiverV)
    where
      senderV :: Value
      senderV = valuePaidTo (eInfoSenderAddr ei) info

      receiverV :: Value
      receiverV = valuePaidTo (toAddress rAddr) info

We need the Script Context info for reading the value that is being paid in
this transaction, and the validator parameter for knowing the receiver address.
Notice that we use the `business logic` function :code:`valueToSender` for
computing the (minimum) value that should be paid to the sender.

Validator Module
----------------

The content of the :code:`Validator` module is mainly boilerplate. It corresponds to the compilation
of the validator and minting policy, from Haskell to Plutus.

For compiling the minting policy, we need to convert the boolean function
:code:`mkControlTokenMintingPolicy` into a compiled :code:`MintingPolicy`.


.. code:: haskell

   controlTokenMP :: ScriptAddress -> MintingPolicy
   controlTokenMP saddr =
       mkMintingPolicyScript $
       $$(compile [|| mkUntypedMintingPolicy . mkControlTokenMintingPolicy ||])
       `applyCode`
       liftCode saddr

Whithout going into details, :code:`controlTokenMP` compiles our boolean function
to Plutus, obtaining a :code:`MintingPolicy`. For that, it first generates an `untyped` version
of our function, and then compiles it. Given that :code:`mkControlTokenMintingPolicy`
receives a parameter, it must be compiled too, by calling :code:`liftCode` function.

Obtaining the resulting currency symbol is straightforward

.. code:: haskell

   controlTokenCurrency :: ScriptAddress -> CurrencySymbol
   controlTokenCurrency = scriptCurrencySymbol . controlTokenMP


Let's now review how to compile the main validator. It's slightly different
to the minting policy. First we need to indicate which types correspond to
Datum and Redeemer, by defining an empty data type and then instantiating
:code:`ValidatorTypes` typeclass
   
.. code:: haskell

   data Escrowing
   instance ValidatorTypes Escrowing where
       type instance DatumType    Escrowing = EscrowDatum
       type instance RedeemerType Escrowing = EscrowRedeemer

Then we compile our boolean function :code:`mkEscrowValidator` to
a :code:`TypedValidator`

.. code:: haskell

   escrowInst :: ReceiverAddress -> TypedValidator Escrowing
   escrowInst raddr =
       mkTypedValidator @Escrowing
       ($$(compile [|| mkEscrowValidator ||])
           `applyCode`
           liftCode raddr
       )
       $$(compile [|| mkUntypedValidator @EscrowDatum @EscrowRedeemer ||])

Finally we obtain the :code:`Validator` and :code:`ScriptAddress`, that are needed
in the off-chain code for building the transactions

.. code:: haskell

   escrowValidator :: ReceiverAddress -> Validator
   escrowValidator = validatorScript . escrowInst

   escrowAddress :: ReceiverAddress -> ScriptAddress
   escrowAddress = mkValidatorAddress . escrowValidator

