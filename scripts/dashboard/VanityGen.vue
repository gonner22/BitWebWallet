<script setup>
import fire from '../../assets/fire.svg';
import pLogo from '../../assets/p_logo.svg';
import { ALERTS, translation, tr } from '../i18n.js';
import { ref, computed, watch } from 'vue';
import { cChainParams } from '../chain_params.js';
import { MAP_B58, createAlert } from '../misc.js';

const addressPrefix = ref('');
const addressPrefixShow = ref(false);
const addressPrefixElement = ref({});
const isGenerating = ref(false);
const attempts = ref(0);
/**
 * @type {Worker[]}
 */
const arrWorkers = [];
const prefixNetwork = computed(() =>
    cChainParams.current.PUBKEY_PREFIX.join(' or ')
);

const emit = defineEmits(['import-wallet']);

watch(addressPrefix, (newValue, oldValue) => {
    if (newValue.length > oldValue.length) {
        const char = newValue.charAt(newValue.length - 1);
        if (!MAP_B58.toLowerCase().includes(char.toLowerCase())) {
            createAlert(
                'warning',
                tr(ALERTS.UNSUPPORTED_CHARACTER, [{ char }]),
                3500
            );
        }
    }
});

function stop() {
    while (arrWorkers.length) {
        const worker = arrWorkers.pop();
        worker.terminate();
    }
    attempts.value = 0;
    isGenerating.value = false;
}

function generate() {
    if (isGenerating.value) return;

    if (typeof Worker === 'undefined')
        return createAlert('error', ALERTS.UNSUPPORTED_WEBWORKERS, 7500);
    if (!addressPrefixShow.value || addressPrefix.value.length === 0) {
        addressPrefixShow.value = true;
        addressPrefixElement.value.focus();
        return;
    }

    // Remove space from prefix
    addressPrefix.value = addressPrefix.value.replace(/ /g, '');
    const prefix = addressPrefix.value.toLowerCase();
    for (const char of prefix) {
        if (!MAP_B58.toLowerCase().includes(char))
            return createAlert(
                'warning',
                tr(ALERTS.UNSUPPORTED_CHARACTER, [{ char: char }]),
                3500
            );
    }

    isGenerating.value = true;
    const nThreads = Math.max(
        Math.floor(window.navigator.hardwareConcurrency * 0.75),
        1
    );
    console.log('Spawning ' + nThreads + ' vanity search threads!');
    for (let i = 0; i < nThreads; i++) {
        const worker = new Worker(
            new URL('../vanitygen_worker.js', import.meta.url)
        );

        const checkResult = ({ data }) => {
            attempts.value++;
            if (data.pub.substr(1, prefix.length).toLowerCase() === prefix) {
                try {
                    emit('import-wallet', data.priv);
                    console.log(
                        `VANITY: Found an address after ${attempts.value} attempts!`
                    );
                } finally {
                    // Stop search even if import fails
                    stop();
                }
            }
        };
        worker.onmessage = checkResult;
        worker.postMessage(cChainParams.current.name);
        arrWorkers.push(worker);
    }
}
</script>

<style>
.v-enter-active,
.v-leave-active {
    transition: opacity 0.3s ease;
}

.v-enter-from,
.v-leave-to {
    opacity: 0;
}
</style>
<template>
    <div class="col-12 col-lg-6 p-2">
        <div class="h-100 dashboard-item dashboard-display">
            <div class="container">
                <div class="coinstat-icon" v-html="fire"></div>

                <div class="col-md-12 dashboard-title">
                    <h3 class="b1t-bold-title" style="font-size: 38px">
                        <span data-i18n="dCardTwoTitle">Create a new</span>
                        <div data-i18n="dCardTwoSubTitle">Vanity Wallet</div>
                    </h3>
                    <p data-i18n="dCardTwoDesc">
                        Create a wallet with a custom prefix, this can take a
                        long time!
                    </p>
                    <span style="opacity: 0.75; font-size: small"
                        ><span data-i18n="vanityPrefixNote"
                            >Note: addresses will always start with:</span
                        >
                        <b>&hairsp; {{ prefixNetwork }}</b></span
                    >
                </div>

                <Transition>
                    <input
                        v-show="addressPrefixShow"
                        v-model="addressPrefix"
                        :disabled="isGenerating"
                        ref="addressPrefixElement"
                        class="center-text"
                        type="text"
                        data-i18n="vanityPrefixInput"
                        placeholder="Address Prefix"
                        maxlength="5"
                        data-testid="prefixInput"
                    />
                </Transition>

                <button
                    class="b1t-button-big"
                    @click="isGenerating ? stop() : generate()"
                    data-testid="generateBtn"
                >
                    <span class="buttoni-icon" v-html="pLogo"> </span>

                    <span class="buttoni-text">
                        <span v-if="isGenerating">
                            <!-- TODO: translate this string -->
                            STOP (SEARCHED
                            {{ attempts.toLocaleString('en-gb') }} KEYS)
                        </span>
                        <span v-else>
                            {{ translation.dCardTwoButton }}
                        </span>
                    </span>
                </button>
            </div>
        </div>
    </div>
</template>
