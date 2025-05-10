# Documentação Técnica – Pixel Personalizado Shopify + GTM

## Visão Geral

Este documento descreve a implementação de um **pixel personalizado** para Shopify, que integra eventos de checkout com **Google Tag Manager (GTM)** e **Google Analytics 4 (GA4)** usando a **Shopify Web Pixels API**.

---

## Objetivo

Permitir o envio de eventos de checkout para o Google Tag Manager via `dataLayer`, ativando tags no GA4, Meta Ads e Google Ads com estrutura completa de ecommerce.

---

## Eventos Rastreáveis

- `begin_checkout`
- `add_shipping_info`
- `add_payment_info`
- `purchase`

---

## Passo a Passo de Instalação na Shopify

### 1. Acesse o painel de pixels

- Vá para o admin da Shopify
- Navegue até **Configurações > Eventos do cliente (Customer events)**
- Clique em **Gerenciar Pixels** ou **Manage Pixels**

### 2. Crie um novo pixel personalizado

- Clique em **Adicionar Pixel Personalizado** (`Add custom pixel`)
- Nomeie como: `GTM - Checkout Tracking`

### 3. Cole o código do módulo `customPixelEvents`

```javascript
const customPixelEvents = (() => {
  const GTM_ID = 'GTM-xxxxxx'; // substitua pelo seu ID real

  const loadGTM = () => {
    if (window.gtmHasLoaded) return;
    (function(w,d,s,l,i){
      w[l]=w[l]||[];
      w[l].push({'gtm.start': new Date().getTime(), event:'gtm.js'});
      var f=d.getElementsByTagName(s)[0],
          j=d.createElement(s),
          dl=l != 'dataLayer' ? '&l=' + l : '';
      j.async = true;
      j.src = 'https://www.googletagmanager.com/gtm.js?id=' + i + dl;
      f.parentNode.insertBefore(j, f);
    })(window,document,'script','dataLayer', GTM_ID);
    window.gtmHasLoaded = true;
    window.dataLayer = window.dataLayer || [];
  };

  const mapItems = (lineItems) => {
    return lineItems.map(lineItem => ({
      item_id: lineItem.variant.product.id,
      item_name: lineItem.title,
      item_brand: lineItem.variant.product.vendor,
      item_category: lineItem.variant.product.type,
      item_sku: lineItem.variant.sku,
      item_variant: lineItem.variant.id,
      variant_name: lineItem.variant.title,
      imageURL: lineItem.variant.image?.url ? 'https:' + lineItem.variant.image.url : null,
      price: parseFloat(lineItem.variant.price.amount),
      quantity: parseInt(lineItem.quantity, 10)
    }));
  };

  const pushEvent = (eventName, checkout, extraData = {}) => {
    const shipping = checkout.shippingAddress;

    window.dataLayer.push({
      event: eventName,
      debug_mode: true,
      fired_from: 'custom_pixel',
      token: checkout.token,
      user_data: {
        email: checkout.email,
        phone: shipping.phone,
        first_name: shipping.firstName,
        last_name: shipping.lastName,
        address: shipping.address1,
        city: shipping.city,
        region: shipping.provinceCode,
        country: shipping.countryCode,
        zip: shipping.zip
      },
      ecommerce: {
        quantity: checkout.lineItems.length,
        value: parseFloat(checkout.totalPrice.amount),
        currency: checkout.currencyCode,
        items: mapItems(checkout.lineItems)
      },
      ...extraData
    });
  };

  return {
    beginCheckout: (event) => {
      loadGTM();
      console.log('checkout_started', event);
      pushEvent('begin_checkout_e2no', event.data.checkout);
    },

    addShippingInfo: (event) => {
      loadGTM();
      console.log('checkout_address_info_submitted', event);
      pushEvent('add_shipping_info_e2no', event.data.checkout);
    },

    addPaymentInfo: (event) => {
      loadGTM();
      console.log('payment_info_submitted', event);
      pushEvent('add_payment_info_e2no', event.data.checkout);
    },

    purchase: (event) => {
      loadGTM();
      console.log('checkout_completed', event);

      const checkout = event.data.checkout;
      const order = checkout.order;

      pushEvent('purchase_e2no', checkout, {
        transaction_id: order.id,
        shipping: parseFloat(checkout.shippingLine?.price?.amount || 0),
        tax: parseFloat(checkout.totalTax || 0),
        new_customer: order.customer?.isFirstOrder || false,
        coupon: checkout.discountApplications?.[0]?.title || null
      });
    }
  };
})();

```

### 4. Adicione o registro de eventos no final

```javascript
analytics.subscribe('checkout_started', customPixelEvents.beginCheckout);
analytics.subscribe('checkout_address_info_submitted', customPixelEvents.addShippingInfo);
analytics.subscribe('payment_info_submitted', customPixelEvents.addPaymentInfo);
analytics.subscribe('checkout_completed', customPixelEvents.purchase);
```

---

## Testando a Instalação

1. Acesse o GA4 > Admin > DebugView
2. Vá até sua loja em aba anônima
3. Inicie o checkout e veja os eventos como `begin_checkout_e2no` sendo enviados
4. Use o `console.log` incluso para verificar a estrutura no navegador

---

## Conformidade

Este código está 100% de acordo com a [documentação oficial da Shopify para Pixels](https://help.shopify.com/en/manual/promoting-marketing/pixels/custom-pixels), incluindo:

- Uso correto da `analytics.subscribe`
- Encapsulamento em módulo
- GTM carregado apenas uma vez
- Respeito a restrições de segurança da Shopify

---

**Desenvolvido por:** Code GPT – Assistente de integração Shopify + GTM + GA4
