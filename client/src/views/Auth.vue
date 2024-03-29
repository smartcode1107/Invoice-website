<template>
    <div class="h-perfectly-centerd h-fullscreen c-auth">
        <div>
            <div class="c-auth__other-view" @click="isSigningUp = !isSigningUp">{{ otherView }}</div>
            <div class="c-auth-system">
                <form name="auth-form" @submit.prevent="handleSubmit">
                    <div class="c-auth-system__title">
                        <h2>{{ currentView }}</h2>
                    </div>

                    <div class="c-auth-system__content">
                        <div class="c-row">
                            <label class="c-row__field" for="email">Email</label>
                            <input v-model="formData.email" class="c-row__input" id="email" name="email" type="email">
                        </div>

                        <div class="c-row">
                            <label class="c-row__field" for="password">Password</label>
                            <input v-model="formData.password" class="c-row__input" id="password" name="password" type="password">
                        </div>
                    </div>
                    
                    <button type="submit" class="c-auth-system__submit">Submit</button>
                </form>
            </div>
        </div>

        <div class="c-demo-session" v-if="demoMessage">
            <div>Since this is just a demo version of the app, once every 5 days all the accumulated data will be cleaned up.</div>

            <div>
                <span class="c-demo-session__message">{{ demoMessage }}</span> until the next clean-up.
            </div>
        </div>
    </div>
</template>

<script>

import { mapActions, mapState } from 'vuex';
import { getMsUntilNextEvent, formatDateFromMs } from '../utils/date';

const CRONJOB_START_DATE = process.env.VUE_APP_CRONJOB_START_DATE;
const WINDOW_SIZE = 5;

export default {

    data: () =>({
        isSigningUp: false,
        formData: {
            email: '',
            password: '',
        },
        remainingMsUntilDemoSessionOver: null,
        demoInformationIntervalId: null,
    }),

    computed: {
        currentView () {
            return this.isSigningUp ? 'Sign Up' : 'Sign In';
        },

        otherView () {
            return this.isSigningUp ? 'Sign In' : 'Sign Up';
        },

        demoMessage () {
            const isDemoSessionDisabled = !CRONJOB_START_DATE;
            if (isDemoSessionDisabled) {
                return null;
            }

            const remainingTime = Object.entries(formatDateFromMs(this.remainingMsUntilDemoSessionOver))
                .reduce((acc, [timeUnit, timeValue]) => acc + (timeValue === 0 ? '' : `${timeValue} ${timeUnit}, `) , ``)
                // Getting rid of the last `, `.
                .slice(0, -2);
            return `${remainingTime}`
        },
    },

    methods: {
        ...mapActions('user', ['attemptLogin', 'attemptRegister']),

        async handleSubmit () {
            if (this.isSigningUp) {
                const attemptSuccessful = await this.attemptRegister(this.formData);

                if (attemptSuccessful) {
                    this.resetForm();
                    this.handleSuccess();
                }

                return;
            }

            const attemptSuccessful = await this.attemptLogin(this.formData);

            if (attemptSuccessful) {
                this.resetForm();
                this.handleSuccess();
            }
        },

        resetForm () {
            this.formData.email = '';
            this.formData.password = '';
        },

        handleSuccess () {
            this.$router.push('/');
        },

        setUpDemoSession () {
            const isDemoSessionDisabled = !CRONJOB_START_DATE;
            if (isDemoSessionDisabled) {
                return;
            }

            const remainingMS = getMsUntilNextEvent(new Date(+CRONJOB_START_DATE), WINDOW_SIZE, new Date());

            this.remainingMsUntilDemoSessionOver = remainingMS;
            this.demoInformationIntervalId = setInterval(() => this.remainingMsUntilDemoSessionOver -= 1000, 1000);

            if (!this.$store.state.endSessionTimeoutId ) {
                const timeoutId = setTimeout(() => {
                    this.$store.commit('SET_END_SESSION_TIMEOUT_ID', null);
                    this.$store.dispatch('user/logout');
                    this.setUpDemoSession();
                }, remainingMS);
                this.$store.commit('SET_END_SESSION_TIMEOUT_ID', timeoutId);
            }
        },
    },

    created () {
        this.setUpDemoSession();
    },

    destroyed () {
        this.demoInformationIntervalId && clearInterval(this.demoInformationIntervalId);
    }
}
</script>

<style lang="scss" scoped>
    $main-blue: #002536;
    $main-white: #FAFAFF;

    .h-visually-hidden {
        width: auto;
        height: auto;
        opacity: 0.00000001;
    }

    .c-auth {
        background-color: #f2f2f2;
        flex-direction: column;

        &__other-view {
            text-align: right;
            color: $main-blue;
            font-style: italic;
            cursor: pointer;
        }
    }

    .c-auth-system {
        position: relative;
        width: 100%;
        min-width: 20rem;
        background: #fff;
        padding: 1rem 1rem 0;
        border-radius: 1px;

        &::before {
            content: "";
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            box-shadow: 0 8px 10px 1px rgba(0, 0, 0, 0.14), 0 3px 14px 2px rgba(0, 0, 0, 0.12), 0 5px 5px -3px rgba(0, 0, 0, 0.2);
            z-index: -1;
        }

        &__title {
            text-align: center;
            color: $main-blue;
        }

        &__content {
            width: 100%;
            padding: 1rem 0;
        }

        &__submit {
            width: 100%;
            height: 1.7rem;
            padding: .3rem;
            letter-spacing: 1px;
            font-size: 1.05rem;
            background-color: $main-blue;
            color: $main-white;
            border: none;
            cursor: pointer;
            margin-bottom: .5rem;
            outline: none;
            box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12), 0 1px 2px rgba(0, 0, 0, 0.24);

            &:hover {
                box-shadow: 0 1px 3px rgba(0, 0, 0, 0.18), 0 4px 2px rgba(0, 0, 0, 0.12);
            }
        }
    }

    .c-row {
        width: 100%;

        &:not(.has-radio) {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        &:not(:first-child) {
            margin-top: 1rem;
        }

        &__field {
            margin-right: .2rem;
            color: $main-blue;
        }

        &__input {
            outline: none;
            border: 1px solid rgba($color: #ccc, $alpha: .3);
            padding: .3rem;
        }

        &.has-radio {
            margin-top: 0;
            margin-bottom: .5rem;
            width: 100%;
            display: flex;
            justify-content: space-around;
        }
        .c-row__radio {
            position: relative;
            font-size: 1.1rem;

            &--content {
                cursor: pointer;
            }

            &--input {
                pointer-events: none;
            }

            &--helper {
                position: absolute;
                top: 0;
                left: -0.25rem;
                cursor: pointer;
                display: block;
                font-size: 1rem;
                user-select: none;
                color: #ccc;

                &::before, &::after {
                    content: '';
                    position: absolute;
                    left: 0;
                    top: 0;
                    margin: 0.25rem;
                    width: .7rem;
                    height: .7rem;
                    transition: transform 0.28s ease;
                    border-radius: 50%;
                    border: 0.125rem solid currentColor;
                }

                &::after {
                    transform: scale(0);
                    border-color: $main-blue;
                    background-color: $main-blue;
                }
            }

            input.c-row__radio--input:checked ~ .c-row__radio--helper::after {
                transform: scale(0.5);
            }
        }
    }

    .c-demo-session {
        text-align: center;
        font-style: italic;
        margin-top: 2rem;

        &__message {
            font-weight: bold;
            color: #002536;
        }
    }
</style>
