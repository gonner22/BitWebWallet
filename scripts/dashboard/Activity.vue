<template>
    <div>
        <center>
            <span class="dcWallet-activityLbl">
                <span :data-i18n="rewards ? 'rewardHistory' : 'activity'">{{ title }}</span>
                <span v-if="rewards"> ({{ rewardsText }} {{ ticker }}) </span>
            </span>
        </center>
        <div class="dcWallet-activity">
            <div class="scrollTable">
                <div>
                    <table class="table table-responsive table-sm stakingTx table-mobile-scroll">
                        <thead>
                            <tr>
                                <th scope="col" class="tx1">{{ translation.time }}</th>
                                <th scope="col" class="tx2">{{ rewards ? translation.ID : translation.description }}</th>
                                <th scope="col" class="tx3">{{ translation.amount }}</th>
                                <th scope="col" class="tx4 text-right"></th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="tx in txs" :key="tx.id">
                                <td class="align-middle pr-10px" style="font-size: 12px">
                                    <i style="opacity: 0.75">{{ tx.date }}</i>
                                </td>
                                <td class="align-middle pr-10px txcode">
                                    <a :href="explorerUrl + '/tx/' + tx.id" target="_blank" rel="noopener noreferrer">
                                        <code class="wallet-code text-center active ptr" style="padding: 4px 9px">{{ tx.content }}</code>
                                    </a>
                                    <span class="copy-icon" @click="copyToClipboard(tx.id)" style="cursor: pointer; margin-left: 5px;">
                                        📋
                                    </span>
                                    <span v-if="copiedTxId === tx.id" class="copied-message" style="font-weight: bold; margin-left: 5px;">
                                        ✔ txid
                                    </span>
                                </td>
                                <td class="align-middle pr-10px">
                                    <b style="font-family: monospace">
                                        <i class="fa-solid" style="padding-right: 3px" :class="[tx.icon]" :style="{ color: tx.colour }"></i>
                                        {{ tx.formattedAmt }} {{ ticker }}
                                    </b>
                                </td>
                                <td class="text-right pr-10px align-middle">
                                    <span class="badge mb-0" :class="{'badge-purple': tx.confirmed, 'bg-danger': !tx.confirmed}">
                                        <i v-if="tx.confirmed" class="fas fa-check"></i>
                                        <i v-else class="fas fa-hourglass-end"></i>
                                    </span>
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>
                <center>
                    <button v-if="!isHistorySynced" class="b1t-button-medium" @click="update(10)">
                        <span class="buttoni-icon">
                            <i class="fas fa-sync fa-tiny-margin" :class="{ 'fa-spin': updating }"></i>
                        </span>
                        <span class="buttoni-text">{{ translation.loadMore }}</span>
                    </button>
                </center>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue';
import { getNetwork } from '../network.js';
import { wallet } from '../wallet.js';
import { mempool } from '../global.js';
import { COIN, cChainParams } from '../chain_params.js';
import { translation } from '../i18n.js';
import { Database } from '../database.js';
import { HistoricalTx, HistoricalTxType } from '../mempool';
import { getNameOrAddress } from '../contacts-book.js';
import { getEventEmitter } from '../event_bus';

const props = defineProps({
    title: String,
    rewards: Boolean,
});

const txs = ref([]);
let txCount = 0;
const updating = ref(false);
const isHistorySynced = ref(false);
const rewardsText = computed(() => `${isHistorySynced.value ? '' : '≥'}${rewardAmount.value.toFixed(2)}`);
const rewardAmount = ref(0);
const ticker = computed(() => cChainParams.current.TICKER);
const explorerUrl = ref(getNetwork()?.strUrl);
const copiedTxId = ref(null); // Nueva referencia para almacenar el ID del TX copiado

const txMap = computed(() => {
    return {
        [HistoricalTxType.STAKE]: {
            icon: 'fa-gift',
            colour: 'white',
            content: translation.activityBlockReward,
        },
        [HistoricalTxType.SENT]: {
            icon: 'fa-minus',
            colour: '#f93c4c',
            content: translation.activitySentTo,
        },
        [HistoricalTxType.RECEIVED]: {
            icon: 'fa-plus',
            colour: '#5cff5c',
            content: translation.activityReceivedWith,
        },
        [HistoricalTxType.DELEGATION]: {
            icon: 'fa-snowflake',
            colour: 'white',
            content: translation.activityDelegatedTo,
        },
        [HistoricalTxType.UNDELEGATION]: {
            icon: 'fa-fire',
            colour: 'white',
            content: translation.activityUndelegated,
        },
        [HistoricalTxType.UNKNOWN]: {
            icon: 'fa-question',
            colour: 'white',
            content: translation.activityUnknown,
        },
    };
});

async function update(txToAdd = 0) {
    const cNet = getNetwork();
    if (!cNet || !cNet.fullSynced) {
        return;
    }

    explorerUrl.value = cNet?.strUrl;

    if (updating.value) return;
    let newTxs = [];

    updating.value = true;

    if (txCount < 10 && txToAdd == 0) txToAdd = 10;

    let found = 0;
    const nHeights = Array.from(mempool.orderedTxmap.keys()).sort((a, b) => a - b);
    while (found < txCount + txToAdd) {
        if (nHeights.length == 0) {
            isHistorySynced.value = true;
            break;
        }
        const nHeight = nHeights.pop();
        const txsAtnHeight = mempool.orderedTxmap.get(nHeight).filter((tx) => {
            return props.rewards ? tx.isCoinStake() : true;
        });
        newTxs = newTxs.concat(txsAtnHeight);
        found += txsAtnHeight.length;
    }
    const arrTXs = wallet.toHistoricalTXs(newTxs);
    await parseTXs(arrTXs);
    txCount = found;
    updating.value = false;
}

watch(translation, async () => await update());

async function parseTXs(arrTXs) {
    const newTxs = [];
    const cNet = getNetwork();

    const dateOptions = {
        year: '2-digit',
        month: '2-digit',
        day: '2-digit',
    };
    const timeOptions = {
        hour: '2-digit',
        minute: '2-digit',
        hour12: true,
    };
    let prevDateString = '';
    let prevTimestamp = 0;
    const cDB = await Database.getInstance();
    const cAccount = await cDB.getAccount();

    for (const cTx of arrTXs) {
        const dateTime = new Date(cTx.time * 1000);
        let strDate =
            Date.now() / 1000 - cTx.time > 86400
                ? ''
                : dateTime.toLocaleTimeString(undefined, timeOptions);
        if (!strDate) {
            if (prevDateString && prevTimestamp - cTx.time * 1000 < 12 * 60 * 60 * 1000) {
                strDate = prevDateString;
            } else {
                prevDateString = dateTime.toLocaleDateString(undefined, dateOptions);
                strDate = prevDateString;
            }
        }
        prevTimestamp = cTx.time * 1000;

        const fConfirmed = cNet.cachedBlockCount - cTx.blockHeight >= (props.rewards ? cChainParams.current.coinbaseMaturity : 6);

        let formattedAmt = '';
        if (cTx.amount < 0.01) {
            formattedAmt = '<0.01';
        } else if (cTx.amount >= 100) {
            formattedAmt = Math.round(cTx.amount).toString();
        } else {
            formattedAmt = cTx.amount.toFixed(2);
        }

        let fSendToSelf = false;
        if (cTx.type === HistoricalTxType.SENT) {
            fSendToSelf = true;
            for (const strAddr of cTx.receivers) {
                if (!wallet.isOwnAddress(strAddr)) {
                    fSendToSelf = false;
                }
            }
        }

        let { icon, colour, content } = txMap.value[cTx.type];
        const match = content.match(/{(.)}/);
        if (match) {
            let who = '';
            if (fSendToSelf) {
                who = translation.activitySelf;
            } else if (cTx.shieldedOutputs) {
                who = translation.activityShieldedAddress;
            } else {
                const arrAddresses = cTx.receivers
                    .map((addr) => [wallet.isOwnAddress(addr), addr])
                    .filter(([isOwnAddress, _]) => {
                        return cTx.type === HistoricalTxType.RECEIVED ? isOwnAddress : !isOwnAddress;
                    })
                    .map(([_, addr]) => getNameOrAddress(cAccount, addr));
                who = [...new Set(arrAddresses.map((addr) => (addr?.length >= 32 ? addr?.substring(0, 6) : addr)))].join(', ') + '...';
            }
            content = content.replace(/{.}/, who);
        }

        newTxs.push({
            date: strDate,
            id: cTx.id,
            content: props.rewards ? cTx.id : content,
            formattedAmt,
            amount: cTx.amount,
            confirmed: fConfirmed,
            icon,
            colour,
        });
    }

    txs.value = newTxs;
}
if (props.rewards) {
    watch(txs, (txs) => (rewardAmount.value = txs.reduce((acc, tx) => acc + tx.amount, 0)));
}

function reset() {
    txs.value = [];
    txCount = 0;
    update(0);
}

function getTxCount() {
    return txCount;
}

function copyToClipboard(txid) {
    const el = document.createElement('textarea');
    el.value = txid;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    copiedTxId.value = txid; // Set ID copied
    setTimeout(() => {
        copiedTxId.value = null; // Reset copied ID after 2 seconds
    }, 2000);
}

getEventEmitter().on('sync-status-update', (_a, _b, done) => done && update());
onMounted(() => update());

defineExpose({ update, reset, getTxCount });
</script>

<style scoped>
/* Add necessary styles here */
.copied-message {
    color: green;
}
</style>
