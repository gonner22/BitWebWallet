<script setup>
import { translation } from '../i18n.js';
import { ref, watch, computed } from 'vue';
import { getAddressColor } from '../contacts-book';
import { promptForContact } from '../contacts-book';
import { sanitizeHTML } from '../misc';

const emit = defineEmits([
    'send',
    'close',
    'max-balance',
    'openQrScan',
    'update:amount',
    'update:address',
]);
// Amount of B1Ts to send in the selected currency (e.g. USD)
const amountCurrency = ref('');
const color = ref('');

const props = defineProps({
    show: Boolean,
    price: Number,
    currency: String,
    amount: String,
    address: String,
});

const address = computed({
    get() {
        return props.address;
    },
    set(value) {
        emit('update:address', value);
        getAddressColor(value).then((c) => (color.value = c));
    },
});
const amount = computed({
    get() {
        return props.amount;
    },
    set(value) {
        emit('update:amount', value.toString());
    },
});

function send() {
    // TODO: Maybe in the future do one of those cool animation that set the
    // Input red
    if (address.value && amount.value)
        emit('send', sanitizeHTML(address.value), amount.value);
}

function syncAmountCurrency() {
    if (amount.value === '') {
        amountCurrency.value = '';
    } else {
        amountCurrency.value = amount.value * props.price;
    }
}

function syncAmount() {
    if (amountCurrency.value === '') {
        amount.value = '';
    } else {
        amount.value = amountCurrency.value / props.price;
    }
}

async function selectContact() {
    address.value = (await promptForContact()) || '';
}
</script>

<template>
    <div v-show="show" class="v-mask">
        <Transition name="transferMenu">
            <div v-show="show" class="exportKeysModalColor transferMenu">
                <div style="padding-top: 5px">
                    <div
                        class="transferExit ptr"
                        @click="
                            show = false;
                            $emit('close');
                        "
                        data-testid="closeButton"
                    >
                        <i class="fa-solid fa-xmark"></i>
                    </div>
                </div>

                <div class="transferBody">
                    <label>{{ translation.address }}</label
                    ><br />

                    <div class="input-group mb-3">
                        <input
                            class="btn-group-input"
                            style="font-family: monospace"
                            :style="{ color }"
                            type="text"
                            :placeholder="translation.receivingAddress"
                            v-model="address"
                            autocomplete="nope"
                        />
                        <div class="input-group-append">
                            <span
                                class="input-group-text ptr"
                                @click="$emit('openQrScan')"
                                ><i class="fa-solid fa-qrcode fa-2xl"></i
                            ></span>
                            <span
                                class="input-group-text ptr"
                                @click="selectContact()"
                                ><i class="fa-solid fa-address-book fa-2xl"></i
                            ></span>
                        </div>
                    </div>

                    <div style="display: none">
                        <label
                            ><span>{{
                                translation.paymentRequestMessage
                            }}</span></label
                        ><br />
                        <div class="input-group">
                            <input
                                class="btn-input"
                                style="font-family: monospace"
                                type="text"
                                disabled
                                placeholder="Payment Request Description"
                                autocomplete="nope"
                            />
                        </div>
                    </div>

                    <label
                        ><span>{{ translation.amount }}</span></label
                    ><br />

                    <div class="row">
                        <div class="col-7 pr-2">
                            <div class="input-group mb-3">
                                <input
                                    class="btn-group-input"
                                    style="padding-right: 0px"
                                    type="number"
                                    placeholder="0.00"
                                    autocomplete="nope"
                                    onkeydown="javascript: return event.keyCode == 69 ? false : true"
                                    data-testid="amount"
                                    @input="$nextTick(syncAmountCurrency)"
                                    v-model="amount"
                                />
                                <div class="input-group-append">
                                    <span class="input-group-text p-0">
                                        <div
                                            @click="$emit('max-balance')"
                                            style="
                                                cursor: pointer;
                                                border: 0px;
                                                border-radius: 7px;
                                                padding: 3px 6px;
                                                margin: 0px 1px;
                                                background: linear-gradient(
                                                    183deg,
                                                    #9621ff9c,
                                                    #7d21ffc7
                                                );
                                                color: #fff;
                                                font-weight: bold;
                                            "
                                        >
                                            {{ translation.sendAmountCoinsMax }}
                                        </div>
                                    </span>
                                    <span class="input-group-text">B1T</span>
                                </div>
                            </div>
                        </div>

                        <div class="col-5 pl-2">
                            <div class="input-group mb-3">
                                <input
                                    class="btn-group-input"
                                    type="text"
                                    placeholder="0.00"
                                    autocomplete="nope"
                                    onkeydown="javascript: return event.keyCode == 69 ? false : true"
                                    data-testid="amountCurrency"
                                    @input="syncAmount"
                                    v-model="amountCurrency"
                                />
                                <div class="input-group-append">
                                    <span class="input-group-text pl-0">{{
                                        currency
                                    }}</span>
                                </div>
                            </div>
                        </div>
                    </div>

                    <div v-if="false">
                        <label
                            ><span>{{ translation.fee }}</span></label
                        ><br />

                        <div class="row text-center">
                            <div class="col-4 pr-1">
                                <div class="feeButton">
                                    Low<br />
                                    9 sat/B
                                </div>
                            </div>

                            <div class="col-4 pl-2 pr-2">
                                <div class="feeButton feeButtonSelected">
                                    Medium<br />
                                    11 sat/B
                                </div>
                            </div>

                            <div class="col-4 pl-1">
                                <div class="feeButton">
                                    High<br />
                                    14 sat/B
                                </div>
                            </div>
                        </div>
                        <br />
                    </div>

                    <div class="text-right pb-2">
                        <button
                            class="b1t-button-medium w-100"
                            style="margin: 0px"
                            @click="send()"
                            data-testid="sendButton"
                        >
                            <span class="buttoni-icon"
                                ><i
                                    class="fas fa-paper-plane fa-tiny-margin"
                                ></i
                            ></span>
                            <span class="buttoni-text">{{
                                translation.send
                            }}</span>
                        </button>
                    </div>
                </div>
            </div>
        </Transition>
    </div>
</template>

<style>
.transferMenu {
    width: calc(100% - 30px);
    position: fixed;
    left: 15px;
    bottom: 0px;
    z-index: 1050;
    border-top-left-radius: 10px;
    border-top-right-radius: 10px;
    min-height: 155px;
    transition: 0.4s;
    /*background-color:rgba(255, 255, 255, 12%)!important;*/
    background-color: #5d2f83c9;
}

@media (min-width: 768px) {
    .transferMenu {
        width: 310px !important;
        left: calc((100% - 310px) / 2) !important;
    }
}

.transferExit {
    position: absolute;
    right: 15px;
}

.transferAnimation {
    transform: translate3d(0, 390px, 0);
}

.transferMenu .transferHeader {
    padding: 9px 12px;
    display: flex;
}

.transferMenu .transferHeader .transferHeaderText {
    width: 100%;
}

.transferItem {
    cursor: pointer;
    margin: 9px 12px;
    display: flex;
}

.transferItem .transferIcon {
    margin-right: 10px;
}

.transferItem .transferText {
    line-height: 17px;
    font-size: 15px;
}

.transferItem .transferText span {
    font-size: 11px;
    color: #dbdbdb;
}

.transferMenu .transferBody {
    padding: 9px 12px;
    font-size: 15px;
}

.transferMenu .transferBody .feeButton {
    background-color: #ffffff00;
    border: 1px solid #ffffff1f;
    border-radius: 8px;
    padding: 5px 0px;
    font-size: 13px;
    cursor: pointer;
    transition: all 0.125s ease-in-out;
}

.transferMenu .transferBody .feeButtonSelected {
    background-color: #ffffff0f;
}

.transferMenu .transferBody .pasteAddress i {
    transition: all 0.125s ease-in-out;
    cursor: pointer;
}

.transferMenu .transferBody .pasteAddress i:hover {
    color: #9621ff9c;
}

.transferMenu-enter-from,
.transferMenu-leave-to {
    transform: translateY(200%);
}

.transferMenu-enter-active .transferMenu-leave-active {
    transition: all 0.3 ease;
}

.v-mask {
    position: fixed;
    z-index: 1050;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    transition: opacity 0.3s ease;
}
</style>
