<template>
  <div class="login-container">
    <el-alert
      v-if="nodeEnv !== 'development'"
      title="beautiful boys and girls 欢迎加入 OPSLI 快速开发平台 QQ群：724850675 | API接口/数据库监控密码均为 admin : 123456 "
      type="success"
      :closable="false"
      style="position: fixed"
    ></el-alert>
    <el-row>
      <el-col :xs="24" :sm="24" :md="12" :lg="16" :xl="16">
        <div style="color: transparent">占位符</div>
      </el-col>
      <el-col :xs="24" :sm="24" :md="12" :lg="8" :xl="8">
        <el-form
          ref="form"
          :model="form"
          :rules="rules"
          class="login-form"
          label-position="left"
        >
          <div class="title">hello !</div>
          <div class="title-tips">欢迎来到{{ title }}！</div>
          <el-form-item style="margin-top: 40px" prop="username">
            <span class="svg-container svg-container-admin">
              <vab-icon :icon="['fas', 'user']" />
            </span>
            <el-input
              v-model.trim="form.username"
              v-focus
              placeholder="请输入用户名"
              tabindex="1"
              type="text"
            />
          </el-form-item>
          <el-form-item prop="password">
            <span class="svg-container">
              <vab-icon :icon="['fas', 'lock']" />
            </span>
            <el-input
              :key="passwordType"
              ref="password"
              v-model.trim="form.password"
              :type="passwordType"
              tabindex="2"
              placeholder="请输入密码"
              @keyup.enter.native="handleLogin"
              show-password
            />
          </el-form-item>

          <el-form-item prop="captcha" v-if="captchaFlag">
            <span class="svg-container">
              <i class="el-icon-warning" />
            </span>
            <el-input
              ref="captcha"
              v-model.trim="form.captcha"
              placeholder="请输入验证码"
              name="captcha"
              type="text"
              tabindex="1"
              autocomplete="on"
              maxlength="10"
              style="width: calc(100% - 200px)"
              @keyup.enter.native="handleLogin"
            />
            <img
              class="captcha"
              style="float: right"
              title="看不起，换一张"
              alt="验证码"
              :src="captchaImg"
              @click="getCaptcha"
            />
          </el-form-item>

          <el-button
            :loading="loading"
            class="login-btn"
            type="primary"
            @click="handleLogin"
          >
            登录
          </el-button>
<!--          <router-link to="/register">-->
<!--            <div style="margin-top: 20px">注册</div>-->
<!--          </router-link>-->
        </el-form>
      </el-col>
    </el-row>
  </div>
</template>

<script>
  import {isNull, isPassword} from "@/utils/validate";
  const { baseURL } = require("@/config/settings");
  export default {
    name: "Login",
    directives: {
      focus: {
        inserted(el) {
          el.querySelector("input").focus();
        },
      },
    },
    data() {
      const validateUsername = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error("用户名不能为空"));
        } else {
          callback();
        }
      };
      const validatePassword = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error('请输入密码'));
        } else if (!isPassword(value)) {
          callback(new Error("密码至少包含大小写字母，数字，且不少于6位"));
        } else {
          callback();
        }
      };
      const validateCaptcha = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error("验证码不能为空"));
        } else {
          callback();
        }
      };

      // UUID
      const uuid = this.uuid();

      return {
        nodeEnv: process.env.NODE_ENV,
        title: this.$baseTitle,
        form: {
          username: "",
          password: "",
          captcha: "",
          uuid: uuid,
        },
        rules: {
          username: [
            {
              required: true,
              trigger: "blur",
              validator: validateUsername,
            },
          ],
          password: [
            {
              required: true,
              trigger: "blur",
              validator: validatePassword,
            },
          ],
          captcha: [
            {
              required: true,
              trigger: "blur",
              validator: validateCaptcha,
            },
          ],
        },
        captchaFlag: false,
        loading: false,
        passwordType: "password",
        redirect: undefined,
        captchaImg: baseURL + "/captcha.jpg?uuid=" + uuid,
      };
    },
    watch: {
      $route: {
        handler(route) {
          this.redirect = (route.query && route.query.redirect) || "/";
        },
        immediate: true,
      },
    },
    created() {
      document.body.style.overflow = "hidden";
    },
    beforeDestroy() {
      document.body.style.overflow = "auto";
    },
    mounted() {
      // 自动登录，
      this.form.username = "demo";
      this.form.password = "Aa123456";
      setTimeout(() => {
        this.handleLogin();
      }, 3000);
    },
    methods: {
      // 获得新验证码
      getCaptcha() {
        this.captchaImg =
          baseURL +
          "/captcha.jpg?uuid=" +
          this.form.uuid +
          "&timestamp=" +
          new Date().getTime();
      },
      // 获取uuid
      uuid() {
        const s = [];
        const hexDigits = "0123456789abcdef";
        for (let i = 0; i < 36; i++) {
          s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
        }
        s[14] = "4";
        s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1);
        s[8] = s[13] = s[18] = s[23];
        this.uuidA = s.join("");
        return this.uuidA;
      },
      handleLogin() {
        this.$refs.form.validate((valid) => {
          if (valid) {
            this.loading = true;
            this.$store
              .dispatch("user/login", this.form)
              .then(() => {
                const routerPath =
                  this.redirect === "/404" || this.redirect === "/401"
                    ? "/"
                    : this.redirect;
                this.$router.push(routerPath).catch(() => {});
                this.loading = false;
              })
              .catch(() => {
                // 获取当前失败次数
                this.$store
                  .dispatch("user/getSlipCount", this.form)
                  .then((ret) => {
                    this.loading = false;
                    const {data} = ret;
                    if(!isNull(data) && data.curr >= data.base ){
                      // 失败次数大于系统规定阈值 开启验证码校验
                      this.captchaFlag = true;
                    }
                  }).catch(() => {
                    this.loading = false;
                  });
              });
          } else {
            return false;
          }
        });
      },
    },
  };
</script>

<style lang="scss" scoped>
  .login-container {
    height: 100vh;
    background: url("~@/assets/login_images/background.jpg") center center fixed
      no-repeat;
    background-size: cover;

    .title {
      font-size: 54px;
      font-weight: 500;
      color: rgba(14, 18, 26, 1);
    }

    .title-tips {
      margin-top: 29px;
      font-size: 26px;
      font-weight: 400;
      color: rgba(14, 18, 26, 1);
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    .login-btn {
      display: inherit;
      width: 220px;
      height: 60px;
      margin-top: 5px;
      border: 0;

      &:hover {
        opacity: 0.9;
      }
    }

    .login-form {
      position: relative;
      max-width: 100%;
      margin: calc((100vh - 425px) / 2) 10% 10%;
      overflow: hidden;

      .forget-password {
        width: 100%;
        margin-top: 40px;
        text-align: left;

        .forget-pass {
          width: 129px;
          height: 19px;
          font-size: 20px;
          font-weight: 400;
          color: rgba(92, 102, 240, 1);
        }
      }
    }

    .tips {
      margin-bottom: 10px;
      font-size: $base-font-size-default;
      color: $base-color-white;

      span {
        &:first-of-type {
          margin-right: 16px;
        }
      }
    }

    .title-container {
      position: relative;

      .title {
        margin: 0 auto 40px auto;
        font-size: 34px;
        font-weight: bold;
        color: $base-color-blue;
        text-align: center;
      }
    }

    .svg-container {
      position: absolute;
      top: 14px;
      left: 15px;
      z-index: $base-z-index;
      font-size: 16px;
      color: #d7dee3;
      cursor: pointer;
      user-select: none;
    }

    .show-password {
      position: absolute;
      top: 14px;
      right: 25px;
      font-size: 16px;
      color: #d7dee3;
      cursor: pointer;
      user-select: none;
    }

    .captcha {
      position: absolute;
      top: -2.5px;
      right: 0;
      width: 180px;
      height: 58px;
      margin-top: 2px;
      cursor: pointer;
    }

    ::v-deep {
      .el-form-item {
        padding-right: 0;
        margin: 20px 0;
        color: #454545;
        background: transparent;
        border: 1px solid transparent;
        border-radius: 2px;

        &__content {
          min-height: $base-input-height;
          line-height: $base-input-height;
        }

        &__error {
          position: absolute;
          top: 100%;
          left: 18px;
          font-size: $base-font-size-small;
          line-height: 18px;
          color: $base-color-red;
        }
      }

      .el-input {
        box-sizing: border-box;

        input {
          height: 58px;
          padding-left: 45px;
          font-size: $base-font-size-default;
          line-height: 58px;
          color: $base-font-color;
          background: #f6f4fc;
          border: 0;
          caret-color: $base-font-color;
        }
      }
    }
  }
</style>
