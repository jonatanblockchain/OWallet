<style scoped>

</style>
<template>
    <div>
    <a-modal 
        :title="$t('nodeStake.signWithWallet')"
        :visible="visible"
        @ok="handleWalletSignOK"
        @cancel="handleWalletSignCancel">
          <div v-if="wallet.key">
              <p>{{$t('nodeStake.enterWalletPass')}}</p>
              <a-input class="input" v-model="walletPassword" :plaecholder="$t('nodeStake.password')" type="password"></a-input>
          </div>
          <div v-if="!wallet.key">
            <div class="font-bold" style="margin-bottom: 10px;">{{$t('ledgerWallet.connectApp')}}</div>
                    <span class="font-medium-black">{{$t('ledgerWallet.status')}}: </span>
                    <span class="font-medium">{{ledgerStatus}} </span>
          </div>
    </a-modal>
    </div>
</template>
<script>
import {mapState} from 'vuex'
import {DEFAULT_SCRYPT} from '../../../core/consts'
import {getNodeUrl} from '../../../core/utils'
import {legacySignWithLedger} from '../../../core/ontLedger'
import {Crypto, TransactionBuilder, TxSignature, utils} from 'ontology-ts-sdk'
export default {
    name: 'SignSendTx',
    props:['tx', 'wallet', 'visible'],
    mounted(){
        if(!this.wallet.key) {//common wallet
            this.$store.dispatch('getLedgerStatus')
        }
    },
    beforeDestroy() {
        clearInterval(this.intervalId);
        this.$store.dispatch('stopGetLedgerStatus')
    },
    data(){
        return {
            walletPassword: ''
        }
    },
    computed:{
        ...mapState({
            ledgerStatus: state => state.LedgerConnector.ledgerStatus,
            ledgerPk : state => state.LedgerConnector.publicKey,
            ledgerWallet: state => state.LedgerConnector.ledgerWallet,
        })
    },
    methods: {
        handleWalletSignCancel() {
            this.walletPassword = '';
            this.$emit('signClose')
        },
        handleWalletSignOK() {
            const tx = this.tx;
            if (this.wallet.key && !this.walletPassword) {
                //common wallet
                this.$message.error(this.$t("nodeStake.passwordEmpty"));
                return;
            }
            if (this.wallet.key) {
                this.$store.dispatch("showLoadingModals");
                const enc = new Crypto.PrivateKey(this.wallet.key);
                let pri;
                try {
                    pri = enc.decrypt(
                        this.walletPassword,
                        new Crypto.Address(this.wallet.address),
                        this.wallet.salt,
                        DEFAULT_SCRYPT
                    );
                } catch (err) {
                    console.log(err);
                    this.$store.dispatch("hideLoadingModals");
                    this.$message.error(this.$t("common.pwdErr"));
                    return;
                }
                TransactionBuilder.signTransaction(tx, pri);
                this.sendTx(tx);
            } else {
                //ledger sign
                if (this.ledgerWallet.address) {
                    this.$store.dispatch("showLoadingModals");
                    const pk = new Crypto.PublicKey(this.ledgerWallet.publicKey);
                    const txSig = new TxSignature();
                    txSig.M = 1;
                    txSig.pubKeys = [pk];
                    tx.payer = new Crypto.Address(this.ledgerWallet.address);;
                    const txData = tx.serializeUnsignedData();
                    legacySignWithLedger(txData).then(res => {
                        // console.log('txSigned: ' + res);
                        const sign = "01" + res; //ECDSAwithSHA256
                        txSig.sigData = [sign];
                        tx.sigs = [txSig];
                        this.sendTx(tx);
                        },
                        err => {
                            this.sending = false;
                            this.ledgerStatus = "";
                            alert(err.message || 'Sign with ledger failed.');
                        }
                    );
                } else {
                    this.$store.dispatch("hideLoadingModals");
                    this.$message.warning(this.$t("ledgerWallet.connectApp"));
                }
            }
            },
            sendTx(tx){
                this.walletPassword = '';
                const url = getNodeUrl();
                const restClient = new Ont.RestClient(url);
                restClient.sendRawTransaction(tx.serialize()).then(res => {
                console.log(res)
                if (res.Error === 0) {
                    this.$message.success(this.$t('common.transSentSuccess'))
                } else if (res.Error === -1) {
                    this.$message.error(this.$t('common.ongNoEnough'))
                    return;
                } else {
                    this.$message.error(res.Result)
                    return;
                }
                this.$emit('txSent')
                const title = this.$t('common.transSentSuccess')
                setTimeout(() => {
                    this.$success({
                        title: title,
                        content: 'Transaction hash: ' + utils.reverseHex(tx.getHash())
                    })
                }, 100)
                })
            },
    }
}
</script>
