> transaction inclusion proof



code :

src/api_server/rpc_server/rpc_trait.rs
```rs
#[rpc]
pub trait Rpc {
    ...
    #[rpc(name = "tx_submit", returns = "TxHash")]
    fn tx_submit(
        &self,
        tx: Box<ZkSyncTx>,
        signature: Box<TxEthSignatureVariant>,
        fast_processing: Option<bool>,
    ) -> FutureResp<TxHash>;
    ...
}
```

core/lib/type/src/tx/zksync_tx.rs
```rs

/// Represents transaction with the corresponding Ethereum signature and the message.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedZkSyncTx {
    /// Underlying zkSync transaction.
    pub tx: ZkSyncTx,
    /// `eth_sign_data` is a tuple of the Ethereum signature and the message
    /// which user should have signed with their private key.
    /// Can be `None` if the Ethereum signature is not required.
    pub eth_sign_data: Option<EthSignData>,
}

/// A set of L2 transaction supported by the zkSync network.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum ZkSyncTx {
    Transfer(Box<Transfer>),
    Withdraw(Box<Withdraw>),
    #[doc(hidden)]
    Close(Box<Close>),
    ChangePubKey(Box<ChangePubKey>),
    ForcedExit(Box<ForcedExit>),
    MintNFT(Box<MintNFT>),
    Swap(Box<Swap>),
    WithdrawNFT(Box<WithdrawNFT>),
}

impl From<Transfer> for ZkSyncTx {
    fn from(transfer: Transfer) -> Self {
        Self::Transfer(Box::new(transfer))
    }
}

impl ZkSyncTx {
    /// Returns the hash of the transaction.
    pub fn hash(&self) -> TxHash {
        let bytes = self.get_bytes();
        let hash = sha256(&bytes);
        let mut out = [0u8; 32];
        out.copy_from_slice(&hash);
        TxHash { data: out }
    }

```
core/lib/types/src/tx/trasfer.rs
```rs
/// `Transfer` transaction performs a move of funds from one zkSync account to another.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Transfer {
    /// zkSync network account ID of the transaction initiator.
    pub account_id: AccountId,
    /// Address of account to transfer funds from.
    pub from: Address,
    /// Address of account to transfer funds to.
    pub to: Address,
    /// Type of token for transfer. Also represents the token in which fee will be paid.
    pub token: TokenId,
    /// Amount of funds to transfer.
    #[serde(with = "BigUintSerdeAsRadix10Str")]
    pub amount: BigUint,
    /// Fee for the transaction.
    #[serde(with = "BigUintSerdeAsRadix10Str")]
    pub fee: BigUint,
    /// Current account nonce.
    pub nonce: Nonce,
    /// Time range when the transaction is valid
    /// This fields must be Option<...> because of backward compatibility with first version of ZkSync
    #[serde(flatten)]
    pub time_range: Option<TimeRange>,
    /// Transaction zkSync signature.
    pub signature: TxSignature,
    #[serde(skip)]
    cached_signer: VerifiedSignatureCache,
}
```

/core/lib/types/src/operations/transfer_op.rs
```rs
/// Transfer operation. For details, see the documentation of [`ZkSyncOp`](./operations/enum.ZkSyncOp.html).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransferOp {
    pub tx: Transfer,
    pub from: AccountId,
    pub to: AccountId,
}
...
fn parse_pub_data(
        bytes: &[u8],
        token_bit_width: usize,
        chunk_bytes: usize,
    ) -> Result<Self, TransferOpError> {
    ...
    Ok(Self {
                tx: Transfer::new(
                    AccountId(from_id),
                    from_address,
                    to_address,
                    TokenId(token),
                    amount,
                    fee,
                    Nonce(nonce),
                    time_range,
                    None,
                ),
                from: AccountId(from_id),
                to: AccountId(to_id),
            })
    ...
```


src/api_server/rpc_server/rpc_impl.rs
```rs
    pub async fn _impl_tx_submit(
        self,
        tx: Box<ZkSyncTx>,
        signature: Box<TxEthSignatureVariant>,
        fast_processing: Option<bool>,
    ) -> Result<TxHash> {
        let start = Instant::now();
        let result = self
            .tx_sender
            .submit_tx(*tx, *signature, fast_processing)
            .await
            .map_err(Error::from);
        metrics::histogram!("api.rpc.tx_submit", start.elapsed());
        result
    }
```
src/api_server/tx_sender.rs
```rs
    pub fn new(
        connection_pool: ConnectionPool,
        ...
    ) -> Self {
        Self::with_client(
            ...
            connection_pool,
            ...
        )
    }

    pub(crate) fn with_client(
        ...
        connection_pool: ConnectionPool,
        ...
    ) -> Self {
        Self {
            ...
            pool: connection_pool,
            ...
        }
    }

    /// Resolves the token from the database.
    pub(crate) async fn token_info_from_id(
        &self,
        token_id: impl Into<TokenLike>,
    ) -> Result<Token, SubmitError> {
        let mut storage = self
            .pool
            .access_storage()
            .await
            .map_err(SubmitError::internal)?;

        self.tokens
            .get_token(&mut storage, token_id)
            .await
            .map_err(SubmitError::internal)?
            // TODO Make error more clean
            .ok_or_else(|| SubmitError::other("Token not found in the DB"))
    }

    /// If `ForcedExit` has Ethereum siganture (e.g. it's a part of a batch), an actual signer
    /// is initiator, not the target, thus, this function will perform a database query to acquire
    /// the corresponding address.
    async fn get_tx_sender(&self, tx: &ZkSyncTx) -> Result<Address, anyhow::Error> {
        match tx {
            ZkSyncTx::ForcedExit(tx) => self.get_address_by_id(tx.initiator_account_id).await,
            _ => Ok(tx.account()),
        }
    }

    async fn get_address_by_id(&self, id: AccountId) -> Result<Address, anyhow::Error> {
        self.pool
            .access_storage()
            .await?
            .chain()
            .account_schema()
            .account_address_by_id(id)
            .await?
            .ok_or_else(|| anyhow::anyhow!("Order signer account id not found in db"))
    }

    pub async fn submit_tx(
        &self,
        mut tx: ZkSyncTx,
        signature: TxEthSignatureVariant,
        fast_processing: Option<bool>,
    ) -> Result<TxHash, SubmitError> {
        ...
        let token = self.token_info_from_id(tx.token_id()).await?;

        let tx_sender = self
            .get_tx_sender(&tx)
            .await
            .or(Err(SubmitError::TxAdd(TxAddError::DbError)))?;

        let verified_tx = verify_tx_info_message_signature(
            &tx,
            tx_sender,
            token.clone(),
            self.get_tx_sender_type(&tx).await?,
            signature.tx_signature().clone(),
            msg_to_sign,
            sign_verify_channel,
        )
        .await?
        .unwrap_tx();

        ...

        // Send verified transactions to the mempool.
        self.core_api_client
            .send_tx(verified_tx)
            .await
            .map_err(SubmitError::communication_core_server)?
            .map_err(SubmitError::TxAdd)?;
    }
```
/core/bin/zksync_api/src/signature_checker.rs
```rs
/// `TxVariant` is used to form a verify request. It is possible to wrap
/// either a single transaction, or the transaction batch.
#[derive(Debug, Clone)]
pub enum TxVariant {
    Tx(SignedZkSyncTx),
    Batch(Vec<SignedZkSyncTx>, Option<EthBatchSignData>),
    Order(Box<Order>),
}

/// Wrapper on a `TxVariant` which guarantees that (a batch of)
/// transaction(s) was checked and signatures associated with
/// this transactions are correct.
///
/// Underlying `TxVariant` is a private field, thus no such
/// object can be created without verification.
#[derive(Debug, Clone)]
pub struct VerifiedTx(TxVariant);
```


src/core_api_client.rs
```
    /// Sends a new transaction to the Core mempool.
    pub async fn send_tx(&self, tx: SignedZkSyncTx) -> anyhow::Result<Result<(), TxAddError>> {
        let endpoint = format!("{}/new_tx", self.addr);
        self.post(&endpoint, tx).await
    }
```
src/apiserver/rest/v1/transactions.rs
```
    async fn send_tx(_tx: Json<SignedZkSyncTx>) -> Json<Result<(), ()>> {
        Json(Ok(()))
    }
```


next is core_api_client.rs. what about tx_sender

- pre tx_sender(offchain):https://zksync.io/api/sdk/js/tutorial.html#checking-zksync-account-balance

```ts
const ethWallet2 = ethers.Wallet.fromMnemonic(MNEMONIC2).connect(ethersProvider);
const syncWallet2 = await zksync.SyncWallet.fromEthSigner(ethWallet2, syncProvider);
const amount = zksync.utils.closestPackableTransactionAmount(ethers.utils.parseEther("0.999"));
const fee = zksync.utils.closestPackableTransactionFee(ethers.utils.parseEther("0.001"));

const transfer = await syncWallet.syncTransfer({
  to: syncWallet2.address(),
  token: "ETH",
  amount,
  fee,
});
```

> link： https://github.com/matter-labs/zksync/tree/master/sdk/zksync.js
```ts
async syncTransfer(transfer: {
        to: Address;
        token: TokenLike;
        amount: BigNumberish;
        fee?: BigNumberish;
        nonce?: Nonce;
        validFrom?: number;
        validUntil?: number;
    }): Promise<Transaction> {
        ...
        const signedTransferTransaction = await this.signSyncTransfer(transfer as any);
        ...
        return submitSignedTransaction(signedTransferTransaction, this.provider);
    }

    async signSyncTransfer(transfer: {
        to: Address;
        token: TokenLike;
        amount: BigNumberish;
        fee: BigNumberish;
        nonce: number;
        validFrom?: number;
        validUntil?: number;
    }): Promise<SignedTransaction> {
        ...
        const signedTransferTransaction = await this.getTransfer(transfer as any);
        ...
        return {
            tx: signedTransferTransaction,
            ethereumSignature
        };
    }

async getTransfer(transfer: {
        to: Address;
        token: TokenLike;
        amount: BigNumberish;
        fee: BigNumberish;
        nonce: number;
        validFrom: number;
        validUntil: number;
    }): Promise<Transfer> {
        if (!this.signer) {
            throw new Error('ZKSync signer is required for sending zksync transactions.');
        }

        const tokenId = this.provider.tokenSet.resolveTokenId(transfer.token);

        const transactionData = {
            accountId: this.accountId,
            from: this.address(),
            to: transfer.to,
            tokenId,
            amount: transfer.amount,
            fee: transfer.fee,
            nonce: transfer.nonce,
            validFrom: transfer.validFrom,
            validUntil: transfer.validUntil
        };

        return this.signer.signSyncTransfer(transactionData);
    }
```

/src/signer.ts
```ts
async signSyncTransfer(transfer: {
        accountId: number;
        from: Address;
        to: Address;
        tokenId: number;
        amount: BigNumberish;
        fee: BigNumberish;
        nonce: number;
        validFrom: number;
        validUntil: number;
    }): Promise<Transfer> {
        const tx: Transfer = {
            ...transfer,
            type: 'Transfer',
            token: transfer.tokenId
        };
        const msgBytes = utils.serializeTransfer(tx);
        const signature = await signTransactionBytes(this.#privateKey, msgBytes);

        return {
            ...tx,
            amount: BigNumber.from(transfer.amount).toString(),
            fee: BigNumber.from(transfer.fee).toString(),
            signature
        };
    }
```



---
ps:

what is mean "match" in rust
```ts
fn main() {
    let t = "abc";
    match t {
        "abc" => println!("Yes"),
        _ => {},
    }
}
```