<script setup lang="ts">
  import { useRouter } from 'vue-router';
  import { authFailure, logIn } from '../data';
  import { ref } from 'vue';
  import moment from 'moment';
  import Captcha from '../components/Captcha.vue';

  const captchaSitekey =
    import.meta.env.VITE_STAIR_HOUSES_CAPTCHA_SITEKEY?.toString() ?? '';

  const router = useRouter();

  const password = ref('');

  const message = ref('');

  const showCaptcha = ref(false);

  let captchaToken = '';

  const logInAction = async () => {
    try {
      if (!password.value) {
        message.value = 'Password cannot be empty';
        return;
      }
      if (showCaptcha.value && captchaSitekey && !captchaToken) {
        message.value = 'Captcha invalid';
        return;
      }
      const result = await logIn(
        password.value,
        showCaptcha.value && captchaSitekey ? captchaToken : undefined
      );
      captchaExpired();
      if (result.success) {
        message.value = '';
        router.push('/admin');
      } else {
        const difference = result.nextTry.getTime() - Date.now();
        message.value =
          'Incorrect password' +
          (difference > 0
            ? ': try again in ' + moment.duration(difference).humanize()
            : '');
        showCaptcha.value = result.showCaptcha;
      }
    } catch (e) {
      console.error(e);
    }
  };

  const captchaVerfiy = (token: string) => {
    captchaToken = token;
  };

  const captchaExpired = () => {
    captchaToken = '';
  };

  authFailure.subscribe(() => {
    router.push('/login');
  });
</script>

<template>
  <div class="login">
    <label class="password-label login-item" for="password"
      >Please enter your password{{ message ? ': ' + message : '' }}</label
    >
    <input
      id="password"
      v-model="password"
      type="password"
      class="password login-item"
      name="password"
      maxlength="20"
      @keyup.enter="logInAction()"
    />
    <Captcha
      v-if="showCaptcha && captchaSitekey"
      :sitekey="captchaSitekey"
      @verify="captchaVerfiy($event)"
      @expired="captchaExpired()"
    ></Captcha>
    <button
      class="login-item login-button"
      @keyup.enter="logInAction()"
      @click="logInAction()"
    >
      Log in
    </button>
  </div>
</template>

<style scoped lang="scss">
  .login {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    width: calc(100% - 1rem);
    height: calc(85% - 1rem);
    padding: 0.5rem;
    margin: 0;
    border: none;
  }

  .login-item {
    margin: 0.5rem;
  }

  .password-label {
    font-size: 1.2rem;
    line-height: 1.2rem;
    padding: 0;
    text-align: center;
    font-weight: bold;
  }

  .password {
    height: 1rem;
    font-size: 1rem;
    line-height: 1rem;
    width: 90%;
    max-width: 40vh;
    padding: 0.25rem;
    border: solid 0.1rem rgb(179, 179, 179);
    border-radius: 1rem;
    box-shadow: 0 0.125rem 0.125rem rgba(0, 0, 0, 0.3);
  }

  .login-button {
    height: 1.9rem;
    font-size: 1rem;
    line-height: 1rem;
    width: 30%;
    max-width: 12vh;
    min-width: 10vw;
    font-weight: bold;
    padding: 0.35rem;
    background-color: #ffffff;
    border: solid 0.05rem rgb(179, 179, 179);
    border-radius: 1rem;
    box-shadow: 0 0.125rem 0.125rem rgba(0, 0, 0, 0.3);
  }

  @media (max-aspect-ratio: 1/1) {
    .login {
      height: calc(100% - 15vw - 1rem);
    }
  }
</style>
