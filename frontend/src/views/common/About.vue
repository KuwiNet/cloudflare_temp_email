<script setup>
import { GithubAlt, Discord, Telegram } from '@vicons/fa'
import { useGlobalState } from '../../store'
const { announcement } = useGlobalState()
</script>

<template>
    <div class="center">
        <n-card :bordered="false" embedded>
            <div v-html="announcement"></div>
            <n-button tag="a" target="_blank" href="https://github.com/KuwiNet/cloudflare_temp_email">
                <template #icon>
                    <n-icon :component="GithubAlt" />
                </template>
                Github
            </n-button>
            <n-button tag="a" target="_blank" href="https://discord.gg/dQEwTWhA6Q">
                <template #icon>
                    <n-icon :component="Discord" />
                </template>
                Discord
            </n-button>
            <n-button tag="a" target="_blank" href="https://t.me/kuwihome">
                <template #icon>
                    <n-icon :component="Telegram" />
                </template>
                Telegram
            </n-button>
        </n-card>
    </div>
</template>

<style scoped>
.center {
    display: flex;
    justify-content: center;
}

.n-card {
    max-width: 800px;
}

.n-button {
    margin-top: 10px;
    margin-left: 10px;
}
</style>
