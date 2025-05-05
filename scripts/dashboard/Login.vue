<script setup>
import fire from '../../assets/fire.svg';
import pLogo from '../../assets/p_logo.svg';
import VanityGen from './VanityGen.vue';
import CreateWallet from './CreateWallet.vue';
import AccessWallet from './AccessWallet.vue';
import { watch, toRefs } from 'vue';

defineEmits(['import-wallet']);

const props = defineProps({
    advancedMode: Boolean,
});
const { advancedMode } = toRefs(props);
</script>

<template>
    <div class="row m-0">
        <CreateWallet
            :advanced-mode="advancedMode"
            @import-wallet="
                (mnemonic, password) =>
                    $emit('import-wallet', {
                        type: 'hd',
                        secret: mnemonic,
                        password,
                    })
            "
        />

        <br />

        <VanityGen
            @import-wallet="
                (wif) => $emit('import-wallet', { type: 'legacy', secret: wif })
            "
        />

        <AccessWallet
            :advancedMode="advancedMode"
            @import-wallet="
                (secret, password) =>
                    $emit('import-wallet', { type: 'hd', secret, password })
            "
        />
    </div>
</template>
