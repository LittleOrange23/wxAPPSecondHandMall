<template>
  <section class="order-wrap">
    <wux-popup position="bottom" :visible="visible" @close="closePopup">
      <div class="pay-wrap">
        <div class="opeartion-wrap">
          <span @click="canclePay">取消</span>
          <span class="title">确认支付</span>
        </div>
        <div class="money-wrap">
          <div class="title">payment</div>
          <div class="money">￥{{ countPrice1 }}</div>
        </div>
        <div class="payee-wrap">
          <span>收款方</span>
          <span>合院二手</span>
        </div>
        <div class="save-wrap">
          <div class="save-btn" @click="openPlain">立即支付</div>
        </div>
      </div>
    </wux-popup>
    <div class="address-box">
      <van-cell
        custom-class="flex-center"
        :title="address.name"
        :value="address.mobile"
        size="large"
        :label="address.address+address.address_detail"
        is-link
        url="/pages/address/main?orderToken=true"
        icon="location-o"
      />
    </div>
    <div class="goods-box">
      <GoodsCart v-for="(cart, cartIndex) in carts" :key="cartIndex" :cart="cart"></GoodsCart>
    </div>
    <div class="order-info-box">
      <van-cell-group>
        <van-cell title="配送方式" value="普通配送"/>
        <van-cell title="运费险" value="卖家送"/>
        <van-cell title="开具发票" value="本次不开具发票"/>
        <van-cell title="订单备注" value="好"/>
      </van-cell-group>
    </div>
    <van-submit-bar
      :price="countPrice"
      button-text="提交订单"
      @submit="onClickButton"
      custom-class="van-submit-bar"
      safe-area-inset-bottom="false"
    />
    <wux-loading id="wux-loading"/>
    <wux-keyboard id="wux-keyboard"/>
  </section>
</template>
<script>
import { getRequest, postRequest } from "@/utils/request.js";
import { $wuxLoading, $wuxKeyBoard } from "../../../static/wux/index.js";
import GoodsCart from "../../components/GoodsCart";

export default {
  components: {
    GoodsCart
  },
  data() {
    return {
      carts: [],
      countPrice: "",
      countPrice1: "",
      openid: "",
      visible: false,
      addressId: "",
      address: {},
      orderId: ""
    };
  },
  created() {
    this.openid = wx.getStorageSync("userinfo").openId;
  },
  mounted() {
    wx.setNavigationBarTitle({
      title: "确认订单"
    });
    this.getCartlist();
    this.getAddress(false);
  },
  onShow() {
    this.addressId = wx.getStorageSync("addressId");
    wx.removeStorageSync("addressId");
    if (this.addressId) {
      this.getAddress(this.addressId);
    }
  },
  methods: {
    async getAddress(id) {
      if (!id) {
        // 默认加载
        const res = await getRequest("/weapp/address/getListAction", {
          openId: this.openid,
          is_default: 1
        });
        this.address = res.data.address;
      } else {
        // 根据id加载
        const res = await getRequest("/weapp/address/detailAction", {
          id: this.addressId
        });
        this.address = res.data.list[0];
      }
    },
    closePopup() {
      this.visible = false;
    },
    showLoading() {
      this.$wuxLoading = $wuxLoading();
      this.$wuxLoading.show({
        text: "订单生成中"
      });
    },
    hideLoading() {
      this.$wuxLoading.hide();
    },
    onClickButton() {
      // this.showLoading()
      // 生成订单 付款
      this.createOrder();
    },
    compentedCountPrice() {
      let count = 0;
      this.carts.forEach(v => {
        count += v.price * v.num;
      });
      this.countPrice = count * 100;
      this.countPrice1 = count;
    },
    async getCartlist() {
      getRequest("/weapp/selectshopcar", {
        openId: wx.getStorageSync("userinfo").openId
      }).then(res => {
        this.carts = res.data.list;
        this.compentedCountPrice();
      });
    },
    // 生成订单
    async createOrder() {
      // 临时解决方案，传送 openid把购物车的商品全部提交，暂时不支持勾选商品
      this.showLoading();
      const res = await postRequest("/weapp/order/create", {
        openid: this.openid,
        addressId: this.address.addressId
      });
      // 失败
      console.log('生成订单', res);
      if (res.data.message == "fail") {
        this.hideLoading();
        wx.showToast({
          title: "失败，发生未知错误",
          icon: "none",
          duration: 2000
        });
      } else {
        // 成功
        // this.countPrice = (res.data.count).toFixed(2)
        this.orderId = res.data.order[0]
        this.hideLoading();
        this.toPay();
      }
    },
    // 弹出支付接口
    async toPay() {
      this.visible = true;
    },
    // 取消支付
    canclePay() {
      this.visible = false;
      wx.showToast({
        title: "取消支付",
        icon: "none",
        duration: 2000
      });
      setTimeout(() => {
        wx.redirectTo({
          url: "/pages/myorder/main"
        });
      }, 500);
    },
    openPlain() {
      const fn = async (title, status) => {
        wx.hideLoading();
        wx.showToast({
          title,
          duration: 2000
        });

        if (status) {
          const res = await postRequest("/weapp/order/editStatus", {
            order_id: this.orderId,
            pay_status: "1", // 已经支付
            trade_status: "1" // 等待发货
          });
          if (res.data.message == "SUCCESS") {
            setTimeout(() => {
              wx.redirectTo({
                url: "/pages/myorder/main"
              });
            }, 1000);
          }
          else {
            wx.showToast({
              title: '发生未知错误联系管理员',
              duration: 2000,
            })
            wx.redirectTo({
              url: '/pages/help/main'
            })
          }
        }
        else {
          setTimeout(() => {
            wx.redirectTo({
              url: '/pages/myorder/main'
            })
          }, 1000);
        }
      };

      $wuxKeyBoard().show({
        className: "className",
        titleText: "安全键盘",
        cancelText: "取消",
        inputText: "输入数字密码",
        showCancel: true,
        disorder: false,
        maxlength: 6,
        callback(value) {
          console.log(`输入的密码是：${value}`);

          wx.showLoading({
            title: "验证支付密码"
          });

          return new Promise((resolve, reject) => {
            // setTimeout(
            //   () =>
            //     Math.ceil(Math.random() * 10) >= 6
            //       ? resolve(fn("密码正确", true))
            //       : reject(fn("密码错误", false)),
            //   2000
            // );
            setTimeout(() => {
              this.visible = false
              if (`${value}` == '123456') {
                resolve(fn('密码正确', true))
              } else {
                reject(fn('密码错误', false))
              }
            }, 2000);
          });
        }
      });
    }
  }
};
</script>
<style lang="less">
.order-wrap {
  background-color: #cccccc;
  .pay-wrap {
    height: 600rpx;
    background-color: #cccccc;
    width: 100%;
    .opeartion-wrap {
      text-align: none;
      height: 100rpx;
      background-color: #37363b;
      color: #ffffff;
      font-size: 36rpx;
      line-height: 100rpx;
      display: flex;
      padding: 0 34rpx;
      .title {
        width: 460rpx;
      }
    }
    .money-wrap {
      height: 200rpx;
      background-color: #f5f5f5;
      color: black;
      font-size: 36rpx;
      display: flex;
      flex-direction: column;
      justify-content: center;
      .money {
        height: 80rpx;
        padding: 10rpx;
        font-size: 60rpx;
      }
    }
    .payee-wrap {
      box-sizing: border-box;
      height: 100rpx;
      padding: 0 40rpx;
      background-color: #ffffff;
      border: 2rpx 0px 2rpx 0pxsolid #d7d7d7;
      display: flex;
      justify-content: space-between;
      align-items: center;
      color: #d1d1d1;
      font-size: 34rpx;
      span:nth-child(2) {
        color: black;
      }
    }
    .save-wrap {
      height: 200rpx;
      background-color: #f5f5f5;
      display: flex;
      justify-content: center;
      align-items: center;
      .save-btn {
        height: 100rpx;
        width: 94%;
        background-color: #1aac19;
        color: #ffffff;
        font-size: 36rpx;
        border-radius: 6rpx;
        text-align: center;
        line-height: 100rpx;
      }
    }
  }
  .address-box {
    background-color: #fff;
    height: 176rpx;
    width: 100%;
    .flex-center {
      align-items: center;
    }
  }
  .goods-box {
    padding-top: 20rpx;
    background: url("http://yanxuan.nosdn.127.net/hxm/yanxuan-wap/p/20161201/style/img/icon-normal/address-bg-bd30f2bfeb.png")
      0 0 repeat-x #fff;
    width: 100%;
    background-color: rgb(100, 240, 119);
  }
  .order-info-box {
    padding: 20rpx 0;
    height: 300rpx;
    width: 100%;
    // background-color: rgb(95, 209, 230);
  }
}
</style>
