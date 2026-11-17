# bayzid

আপনার E-commerce System-এর জন্য এবার আমরা bKash + SSLCommerz Dual Payment Gateway Integration তৈরি করব।

এখানে সবচেয়ে গুরুত্বপূর্ণ বিষয় হলো: Customer Checkout-এ Voucher Apply করার পর Payment সফল না হওয়া পর্যন্ত Voucher-কে সরাসরি `REDEEMED` করা যাবে না। প্রথমে Voucher Reserve করতে হবে, Payment Gateway থেকে Server-side Verification করতে হবে এবং সফল হলে Voucher Redeem করতে হবে।

আমরা এই Architecture অনুসরণ করব:

* Next.js: Checkout UI ও Payment Redirect।

* NestJS: Payment Initiation ও Webhook Handling।

* Prisma: Order, Payment ও Voucher Transaction।

* bKash: Tokenized Checkout।

* SSLCommerz: Hosted Payment Gateway।

* PostgreSQL: Atomic Payment Confirmation ও Idempotency।

### ১. Payment Workflow

Customer Checkout

Order + Voucher Selection

Voucher Reservation

Temporary Lock — not yet redeemed

bKash

Create Payment

SSLCommerz

Session Creation

Server-side Payment Verification

Gateway API + Amount + Transaction ID

Atomic Database Transaction

Payment PAID + Order PAID + Voucher REDEEMED

Payment ব্যর্থ বা বাতিল হলে Voucher Reservation Release হবে, যদি Reservation Expire না হয়ে থাকে এবং অন্য কোনো Payment সেটি ব্যবহার না করে থাকে।

### ২. Prisma Schema Update

আগের Voucher Schema-তে `RESERVED` Status যোগ করতে হবে। পাশাপাশি Payment Tracking-এর জন্য নতুন Model তৈরি করব।

`prisma/schema.prisma`

```prisma

enum VoucherStatus {
  ACTIVE
  RESERVED
  REDEEMED
  EXPIRED
  CANCELLED
}

enum PaymentProvider {
  BKASH
  SSLCOMMERZ
}

enum PaymentStatus {
  INITIATED
  PENDING
  PAID
  FAILED
  CANCELLED
  EXPIRED
  REVIEW_REQUIRED
}

enum OrderStatus {
  PENDING_PAYMENT
  PAID
  PAYMENT_FAILED
  CANCELLED
  REFUNDED
}

model Payment {
  id                String          @id @default(uuid())
  orderId           String
  provider          PaymentProvider
  status            PaymentStatus   @default(INITIATED)

  transactionId     String          @unique
  gatewayPaymentId  String?         @unique

  amount            Decimal         @db.Decimal(12, 2)
  currency          String          @default("BDT")

  paidAt            DateTime?
  createdAt         DateTime        @default(now())
  updatedAt         DateTime        @updatedAt

  order Order @relation(
    fields: [orderId],
    references: [id]
  )

  @@index([orderId, status])
}

model Voucher {
  id                  String        @id @default(uuid())
  code                String        @unique
  userId              String
  campaignId          String        @unique

  discount            Decimal       @db.Decimal(10, 2)
  voucherType         VoucherType
  status              VoucherStatus @default(ACTIVE)

  minOrder            Decimal       @default(0) @db.Decimal(10, 2)
  maxDiscount         Decimal?      @db.Decimal(10, 2)

  expiresAt           DateTime
  reservedOrderId     String?
  reservationExpiresAt DateTime?

  createdAt           DateTime      @default(now())

  redemption VoucherRedemption?

  @@index([userId, status])
  @@index([status, reservationExpiresAt])
}

model Order {
  id        String      @id @default(uuid())
  userId    String
  subtotal  Decimal     @db.Decimal(12, 2)
  discount  Decimal     @default(0) @db.Decimal(12, 2)
  total     Decimal     @db.Decimal(12, 2)
  status    OrderStatus @default(PENDING_PAYMENT)

  payments Payment[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

এখানে ধরে নেওয়া হয়েছে আপনার বিদ্যমান `Order`, `VoucherType` এবং `VoucherRedemption` Model-এর সঙ্গে এই Fields মিলিয়ে নেওয়া হবে। আগের Model-এর Duplicate Definition তৈরি করবেন না।

Migration:

```bash
npx prisma migrate dev --name payment_gateway_integration
npx prisma generate
```

কেন `RESERVED` প্রয়োজন?

ধরুন, একজন Customer ১০% Discount Voucher ব্যবহার করে Checkout করলেন। Payment এখনো Pending। এই সময় অন্য Checkout-এ একই Voucher ব্যবহার করা যাবে না। তবে Payment ব্যর্থ হলে নির্দিষ্ট সময় পর Voucher আবার `ACTIVE` করা যাবে।

### ৩. Environment Configuration

`.env`

```env
# Application
API_URL=https://api.example.com
FRONTEND_URL=https://example.com

# bKash
BKASH_BASE_URL=https://tokenized.pay.bka.sh
BKASH_APP_KEY=your_app_key
BKASH_APP_SECRET=your_app_secret
BKASH_USERNAME=your_username
BKASH_PASSWORD=your_password

# SSLCommerz
SSLCOMMERZ_BASE_URL=https://securepay.sslcommerz.com
SSLCOMMERZ_STORE_ID=your_store_id
SSLCOMMERZ_STORE_PASSWORD=your_store_password

# Sandbox: use sandbox endpoints and credentials
```

Sandbox এবং Live Credentials আলাদা রাখবেন। `.env` কখনো GitHub-এ Commit করবেন না।

### ৪. Payment Initiation Service

প্রথমে Order-এর Amount Server থেকে বের করতে হবে। Customer-এর Browser থেকে পাঠানো `total`, `discount` বা `payableAmount` গ্রহণ করা যাবে না।

`src/payment/payment.service.ts`

নিচে একটি Common Payment Initiation Service:

```typescript

import {
  Injectable,
  BadRequestException,
} from '@nestjs/common';

import { PrismaService } from '../prisma/prisma.service';
import { randomUUID } from 'node:crypto';

@Injectable()
export class PaymentService {
  constructor(
    private readonly prisma: PrismaService,
  ) {}

  async createPayment(
    orderId: string,
    userId: string,
    provider: 'BKASH' | 'SSLCOMMERZ',
    voucherId?: string,
  ) {
    const payment = await this.prisma.$transaction(
      async (tx) => {

        const order = await tx.order.findFirst({
          where: {
            id: orderId,
            userId,
            status: 'PENDING_PAYMENT',
          },
        });

        if (!order) {
          throw new BadRequestException(
            'Invalid order',
          );
        }

        if (Number(order.total) <= 0) {
          throw new BadRequestException(
            'Invalid payment amount',
          );
        }

        // Reserve voucher for 15 minutes

        if (voucherId) {
          const reservation =
            await tx.voucher.updateMany({
              where: {
                id: voucherId,
                userId,
                status: 'ACTIVE',
                expiresAt: {
                  gt: new Date(),
                },
              },
              data: {
                status: 'RESERVED',
                reservedOrderId: orderId,
                reservationExpiresAt:
                  new Date(Date.now() + 15 * 60 * 1000),
              },
            });

          if (reservation.count !== 1) {
            throw new BadRequestException(
              'Voucher unavailable',
            );
          }
        }

        return tx.payment.create({
          data: {
            orderId,
            provider,
            amount: order.total,
            currency: 'BDT',
            transactionId: randomUUID(),
            status: 'INITIATED',
          },
        });
      },
      {
        isolationLevel: 'Serializable',
      },
    );

    return payment;
  }
}
```

এখানে একটি গুরুত্বপূর্ণ বিষয়: Order-এ Voucher Discount ইতিমধ্যে হিসাব করে রাখা থাকতে হবে। `voucherId` দিয়ে Reserve করার পাশাপাশি নিশ্চিত করুন যে Voucher-টি ওই Order-এর Discount Calculation-এর অংশ ছিল। Production-এ `OrderVoucher` অথবা Checkout Reservation Model দিয়ে এই সম্পর্ক সংরক্ষণ করা আরও ভালো।

### ৫. bKash Payment Gateway Integration

bKash Tokenized Checkout-এর সাধারণ Flow:

* Grant Token।

* Create Payment।

* Customer-কে bKash Checkout URL-এ Redirect।

* Callback-এ Payment ID গ্রহণ।

* Execute Payment।

* Query Payment দিয়ে Transaction যাচাই।

* Database-এ Payment Confirm।

bKash-এর API-তে Grant Token, Create Payment, Execute Payment ও Query Payment আলাদা ধাপ। আপনার Merchant Account-এর API Version ও Credentials অনুযায়ী Endpoint নিশ্চিত করবেন। bkash.devarif.me +1

### ৫.১ bKash Service

`src/payment/bkash.service.ts`

```typescript

import {
  Injectable,
  BadGatewayException,
} from '@nestjs/common';

import { ConfigService } from '@nestjs/config';

@Injectable()
export class BkashService {
  private baseUrl: string;

  constructor(
    private readonly config: ConfigService,
  ) {
    this.baseUrl = this.config.getOrThrow(
      'BKASH_BASE_URL',
    );
  }

  private async grantToken(): Promise<string> {
    const response = await fetch(
      `${this.baseUrl}/v1.2.0-beta/tokenized/checkout/token/grant`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Accept: 'application/json',
          username: this.config.getOrThrow(
            'BKASH_USERNAME',
          ),
          password: this.config.getOrThrow(
            'BKASH_PASSWORD',
          ),
          'X-APP-Key': this.config.getOrThrow(
            'BKASH_APP_KEY',
          ),
        },
        body: JSON.stringify({
          app_key: this.config.getOrThrow(
            'BKASH_APP_KEY',
          ),
          app_secret: this.config.getOrThrow(
            'BKASH_APP_SECRET',
          ),
        }),
        signal: AbortSignal.timeout(30000),
      },
    );

    const data = await response.json();

    if (!response.ok || !data.id_token) {
      throw new BadGatewayException(
        'bKash authentication failed',
      );
    }

    return data.id_token;
  }

  async createPayment(
    amount: string,
    invoice: string,
  ) {
    const token = await this.grantToken();

    const response = await fetch(
      `${this.baseUrl}/v1.2.0-beta/tokenized/checkout/create`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Accept: 'application/json',
          Authorization: token,
          'X-APP-Key': this.config.getOrThrow(
            'BKASH_APP_KEY',
          ),
        },
        body: JSON.stringify({
          mode: '0011',
          payerReference: invoice,
          callbackURL:
            `${this.config.getOrThrow('API_URL')}/payments/bkash/callback`,
          amount,
          currency: 'BDT',
          intent: 'sale',
          merchantInvoiceNumber: invoice,
        }),
        signal: AbortSignal.timeout(30000),
      },
    );

    const data = await response.json();

    if (
      !response.ok ||
      !data.paymentID ||
      !data.bkashURL
    ) {
      throw new BadGatewayException(
        'Unable to create bKash payment',
      );
    }

    return data;
  }

  async executePayment(paymentID: string) {
    const token = await this.grantToken();

    const response = await fetch(
      `${this.baseUrl}/v1.2.0-beta/tokenized/checkout/execute`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: token,
          'X-APP-Key': this.config.getOrThrow(
            'BKASH_APP_KEY',
          ),
        },
        body: JSON.stringify({
          paymentID,
        }),
        signal: AbortSignal.timeout(30000),
      },
    );

    const data = await response.json();

    if (!response.ok) {
      throw new BadGatewayException(
        'bKash execution failed',
      );
    }

    return data;
  }

  async queryPayment(paymentID: string) {
    const token = await this.grantToken();

    const response = await fetch(
      `${this.baseUrl}/v1.2.0-beta/tokenized/checkout/payment/status`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: token,
          'X-APP-Key': this.config.getOrThrow(
            'BKASH_APP_KEY',
          ),
        },
        body: JSON.stringify({
          paymentID,
        }),
        signal: AbortSignal.timeout(30000),
      },
    );

    const data = await response.json();

    if (!response.ok) {
      throw new BadGatewayException(
        'bKash payment query failed',
      );
    }

    return data;
  }
}
```

নোট: bKash-এর API Base URL ও Path আপনার Merchant Portal-এ দেওয়া নির্দিষ্ট API Version অনুযায়ী মিলিয়ে নেবেন। বিশেষ করে Query Payment Endpoint-এর Path ও Response Fields Sandbox-এ পরীক্ষা করুন।

### ৫.২ bKash Payment Initiation

```typescript
async initiateBkash(
  orderId: string,
  userId: string,
  voucherId?: string,
) {
  const payment = await this.paymentService.createPayment(
    orderId,
    userId,
    'BKASH',
    voucherId,
  );

  try {
    const result = await this.bkashService.createPayment(
      Number(payment.amount).toFixed(2),
      payment.transactionId,
    );

    await this.prisma.payment.update({
      where: {
        id: payment.id,
      },
      data: {
        status: 'PENDING',
        gatewayPaymentId: result.paymentID,
      },
    });

    return {
      paymentId: payment.id,
      redirectUrl: result.bkashURL,
    };

  } catch (error) {
    // Do not assume the gateway did not create a payment
    // when a network timeout occurs.
    throw error;
  }
}
```

Network Timeout হলে নতুন Payment তৈরি করার আগে পুরোনো Transaction Query করে Status নিশ্চিত করতে হবে।

### ৬. SSLCommerz Integration

SSLCommerz-এর Hosted Checkout Flow:

* Server থেকে Session API Call।

* `GatewayPageURL` গ্রহণ।

* Customer Redirect।

* SSLCommerz IPN গ্রহণ।

* Order Validation API Call।

* Amount, Currency ও Transaction ID যাচাই।

* Payment Confirm।

SSLCommerz-এর অফিসিয়াল Documentation অনুযায়ী IPN পাওয়া গেলেই Payment সফল ধরে নেওয়া যাবে না। অবশ্যই Order Validation API দিয়ে যাচাই করতে হবে। `risk_level = 1` হলে Payment-কে Review-এর জন্য Hold করতে হবে। SSLCOMMERZ +1

Official Documentation: [SSLCommerz API Documentation](https://developer.sslcommerz.com/doc/v4/?utm_source=chatgpt.com)।

### ৬.১ SSLCommerz Service

`src/payment/sslcommerz.service.ts`

```typescript

import {
  Injectable,
  BadGatewayException,
} from '@nestjs/common';

import { ConfigService } from '@nestjs/config';

@Injectable()
export class SSLCommerzService {
  private baseUrl: string;

  constructor(
    private readonly config: ConfigService,
  ) {
    this.baseUrl = this.config.getOrThrow(
      'SSLCOMMERZ_BASE_URL',
    );
  }

  async createSession(params: {
    amount: string;
    transactionId: string;
    customerName: string;
    customerEmail: string;
    customerPhone: string;
  }) {

    const form = new URLSearchParams({
      store_id: this.config.getOrThrow(
        'SSLCOMMERZ_STORE_ID',
      ),

      store_passwd: this.config.getOrThrow(
        'SSLCOMMERZ_STORE_PASSWORD',
      ),

      total_amount: params.amount,
      currency: 'BDT',
      tran_id: params.transactionId,

      success_url:
        `${this.config.getOrThrow('API_URL')}/payments/sslcommerz/success`,

      fail_url:
        `${this.config.getOrThrow('API_URL')}/payments/sslcommerz/fail`,

      cancel_url:
        `${this.config.getOrThrow('API_URL')}/payments/sslcommerz/cancel`,

      ipn_url:
        `${this.config.getOrThrow('API_URL')}/payments/sslcommerz/ipn`,

      cus_name: params.customerName,
      cus_email: params.customerEmail,
      cus_phone: params.customerPhone,

      cus_add1: 'Customer Address',
      cus_city: 'Dhaka',
      cus_country: 'Bangladesh',

      shipping_method: 'NO',
      product_name: 'E-commerce Order',
      product_category: 'General',
      product_profile: 'general',

      value_a: params.transactionId,
    });

    const response = await fetch(
      `${this.baseUrl}/gwprocess/v4/api.php`,
      {
        method: 'POST',
        headers: {
          'Content-Type':
            'application/x-www-form-urlencoded',
        },
        body: form.toString(),
        signal: AbortSignal.timeout(30000),
      },
    );

    const data = await response.json();

    if (
      !response.ok ||
      data.status !== 'SUCCESS' ||
      !data.GatewayPageURL
    ) {
      throw new BadGatewayException(
        'SSLCommerz session creation failed',
      );
    }

    return data;
  }

  async validateTransaction(valId: string) {

    const url = new URL(
      `${this.baseUrl}/validator/api/validationserverAPI.php`,
    );

    url.searchParams.set('val_id', valId);

    url.searchParams.set(
      'store_id',
      this.config.getOrThrow(
        'SSLCOMMERZ_STORE_ID',
      ),
    );

    url.searchParams.set(
      'store_passwd',
      this.config.getOrThrow(
        'SSLCOMMERZ_STORE_PASSWORD',
      ),
    );

    url.searchParams.set('format', 'json');

    const response = await fetch(url, {
      method: 'GET',
      signal: AbortSignal.timeout(30000),
    });

    const data = await response.json();

    if (!response.ok) {
      throw new BadGatewayException(
        'SSLCommerz validation failed',
      );
    }

    return data;
  }
}
```

`cus_add1`, `cus_city` এবং Customer Information-এর Placeholder-গুলো আপনার Order/Customer Database থেকে পূরণ করবেন।

### ৬.২ SSLCommerz Payment Initiation

```typescript
async initiateSSLCommerz(
  orderId: string,
  userId: string,
  customer: {
    name: string;
    email: string;
    phone: string;
  },
  voucherId?: string,
) {
  const payment = await this.paymentService.createPayment(
    orderId,
    userId,
    'SSLCOMMERZ',
    voucherId,
  );

  const session =
    await this.sslCommerzService.createSession({
      amount: Number(payment.amount).toFixed(2),
      transactionId: payment.transactionId,
      customerName: customer.name,
      customerEmail: customer.email,
      customerPhone: customer.phone,
    });

  await this.prisma.payment.update({
    where: {
      id: payment.id,
    },
    data: {
      status: 'PENDING',
      gatewayPaymentId: session.sessionkey ?? null,
    },
  });

  return {
    paymentId: payment.id,
    redirectUrl: session.GatewayPageURL,
  };
}
```

এখানে `sessionkey`-এর পরিবর্তে আপনার ব্যবহৃত SSLCommerz API Response অনুযায়ী Session ID সংরক্ষণ করবেন। Gateway Session ID এবং Merchant Transaction ID আলাদা করে রাখাই ভালো।

### ৭. Central Payment Confirmation Service

এখন সবচেয়ে গুরুত্বপূর্ণ অংশ: bKash এবং SSLCommerz উভয়ের Payment Verification শেষ হলে একই Confirmation Service ব্যবহার করব।

এতে আলাদা Gateway-এর জন্য আলাদা Voucher Redemption Logic লিখতে হবে না।

`src/payment/payment-confirmation.service.ts`

```typescript

import {
  Injectable,
  BadRequestException,
} from '@nestjs/common';

import { PrismaService } from '../prisma/prisma.service';

import { Prisma } from '@prisma/client';

@Injectable()
export class PaymentConfirmationService {
  constructor(
    private readonly prisma: PrismaService,
  ) {}

  async confirmPayment(
    transactionId: string,
    verified: {
      amount: string;
      currency: string;
      gatewayTransactionId: string;
    },
  ) {
    return this.prisma.$transaction(
      async (tx) => {

        const payment = await tx.payment.findUnique({
          where: {
            transactionId,
          },
        });

        if (!payment) {
          throw new BadRequestException(
            'Payment not found',
          );
        }

        // Idempotency: repeated callback/IPN
        if (payment.status === 'PAID') {
          return {
            success: true,
            message: 'Payment already confirmed',
          };
        }

        if (
          !['PENDING', 'INITIATED'].includes(
            payment.status,
          )
        ) {
          throw new BadRequestException(
            'Payment cannot be confirmed',
          );
        }

        // Verify amount and currency

        if (
          new Prisma.Decimal(verified.amount).comparedTo(
            payment.amount,
          ) !== 0 ||
          verified.currency !== payment.currency
        ) {
          throw new BadRequestException(
            'Payment amount or currency mismatch',
          );
        }

        // Mark payment as paid atomically

        const paymentUpdate =
          await tx.payment.updateMany({
            where: {
              id: payment.id,
              status: {
                in: ['PENDING', 'INITIATED'],
              },
            },
            data: {
              status: 'PAID',
              paidAt: new Date(),
            },
          });

        if (paymentUpdate.count !== 1) {
          throw new BadRequestException(
            'Payment already processed',
          );
        }

        // Mark order as paid

        const orderUpdate =
          await tx.order.updateMany({
            where: {
              id: payment.orderId,
              status: 'PENDING_PAYMENT',
            },
            data: {
              status: 'PAID',
            },
          });

        if (orderUpdate.count !== 1) {
          throw new BadRequestException(
            'Order is not available for payment confirmation',
          );
        }

        // Find voucher reserved for this order

        const voucher = await tx.voucher.findFirst({
          where: {
            reservedOrderId: payment.orderId,
            status: 'RESERVED',
          },
        });

        if (voucher) {

          if (
            !voucher.reservationExpiresAt ||
            voucher.reservationExpiresAt <= new Date()
          ) {
            throw new BadRequestException(
              'Voucher reservation expired',
            );
          }

          // Redeem voucher atomically

          const redeemed =
            await tx.voucher.updateMany({
              where: {
                id: voucher.id,
                status: 'RESERVED',
                reservedOrderId: payment.orderId,
                reservationExpiresAt: {
                  gt: new Date(),
                },
              },
              data: {
                status: 'REDEEMED',
                reservedOrderId: null,
                reservationExpiresAt: null,
              },
            });

          if (redeemed.count !== 1) {
            throw new BadRequestException(
              'Voucher redemption failed',
            );
          }

          await tx.voucherRedemption.create({
            data: {
              voucherId: voucher.id,
              orderId: payment.orderId,
              userId: voucher.userId,
              discount: (
                await tx.order.findUniqueOrThrow({
                  where: {
                    id: payment.orderId,
                  },
                  select: {
                    discount: true,
                  },
                })
              ).discount,
            },
          });
        }

        return {
          success: true,
          message: 'Payment confirmed successfully',
          transactionId,
          orderId: payment.orderId,
          voucherRedeemed: Boolean(voucher),
        };
      },
      {
        isolationLevel: 'Serializable',
        timeout: 15000,
      },
    );
  }
}
```

একটি গুরুত্বপূর্ণ Production Rule: Payment Confirmation এবং Order Fulfillment আলাদা রাখতে হবে। Payment Confirm হলেই Order-কে Paid করা যাবে, কিন্তু Stock Deduction, Shipment ও Delivery আলাদা ব্যবসায়িক Workflow-এ সম্পন্ন হবে।

আরও একটি বিষয়: একই Order-এর জন্য একাধিক Payment Attempt থাকলে প্রথম সফল Payment-ই Order Paid করবে। পরবর্তী সফল Payment-কে Duplicate Payment হিসেবে শনাক্ত করে Refund/Manual Review-তে পাঠাতে হবে; সেটি আবার Order বা Voucher পরিবর্তন করতে পারবে না।

### ৮. bKash Callback Verification

bKash-এর Redirect Callback-কে সরাসরি Payment Success হিসেবে বিশ্বাস করবেন না।

Callback-এ পাওয়া `paymentID` দিয়ে:

* নিজের Database-এ Payment খুঁজুন।

* Payment-এর Provider যাচাই করুন।

* bKash Execute API Call করুন।

* Query Payment API দিয়ে Status যাচাই করুন।

* Amount, Currency এবং `trxID` মিলিয়ে নিন।

* সবকিছু সঠিক হলে Central Confirmation Service Call করুন।

`src/payment/bkash-callback.service.ts`

```typescript

import {
  Injectable,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class BkashCallbackService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly bkash: BkashService,
    private readonly confirmation: PaymentConfirmationService,
  ) {}

  async handleCallback(
    paymentID: string,
    callbackStatus?: string,
  ) {
    const payment =
      await this.prisma.payment.findFirst({
        where: {
          gatewayPaymentId: paymentID,
          provider: 'BKASH',
        },
      });

    if (!payment) {
      throw new BadRequestException(
        'Payment not found',
      );
    }

    if (payment.status === 'PAID') {
      return {
        success: true,
        message: 'Already confirmed',
      };
    }

    // Never trust callbackStatus alone.

    if (callbackStatus === 'cancel') {
      return {
        success: false,
        message: 'Payment cancelled by customer',
      };
    }

    // Execute payment

    await this.bkash.executePayment(paymentID);

    // Independently query gateway status

    const result =
      await this.bkash.queryPayment(paymentID);

    if (
      result.transactionStatus !== 'Completed' ||
      !result.trxID ||
      !result.amount ||
      result.currency !== 'BDT'
    ) {
      throw new BadRequestException(
        'bKash payment is not verified',
      );
    }

    return this.confirmation.confirmPayment(
      payment.transactionId,
      {
        amount: result.amount,
        currency: result.currency,
        gatewayTransactionId: result.trxID,
      },
    );
  }
}
```

bKash-এর Response Field এবং Status আপনার Merchant API Version-এর Documentation অনুযায়ী নিশ্চিত করবেন। `Completed` ছাড়া অন্য কোনো Status-এ Order Paid করবেন না।

### ৯. SSLCommerz IPN Webhook Handling

SSLCommerz IPN সবচেয়ে গুরুত্বপূর্ণ অংশগুলোর একটি।

Customer Browser বন্ধ করে দিলেও SSLCommerz Server আপনার API-তে IPN পাঠাতে পারে। তাই শুধু Success Redirect-এর উপর নির্ভর করা যাবে না।

`src/payment/sslcommerz-webhook.service.ts`

```typescript

import {
  Injectable,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class SSLCommerzWebhookService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly ssl: SSLCommerzService,
    private readonly confirmation: PaymentConfirmationService,
  ) {}

  async handleIPN(body: any) {

    if (
      !body.val_id ||
      !body.tran_id
    ) {
      throw new BadRequestException(
        'Invalid IPN payload',
      );
    }

    const payment =
      await this.prisma.payment.findFirst({
        where: {
          transactionId: body.tran_id,
          provider: 'SSLCOMMERZ',
        },
      });

    if (!payment) {
      throw new BadRequestException(
        'Payment not found',
      );
    }

    if (payment.status === 'PAID') {
      return {
        success: true,
        message: 'Already processed',
      };
    }

    // Verify directly with SSLCommerz

    const verified =
      await this.ssl.validateTransaction(
        body.val_id,
      );

    // Check validated transaction details

    if (
      verified.status !== 'VALID' &&
      verified.status !== 'VALIDATED'
    ) {
      throw new BadRequestException(
        'Transaction validation failed',
      );
    }

    if (
      verified.tran_id !== payment.transactionId ||
      verified.currency !== 'BDT'
    ) {
      throw new BadRequestException(
        'Transaction mismatch',
      );
    }

    if (
      Number(verified.amount).toFixed(2) !==
      Number(payment.amount).toFixed(2)
    ) {
      throw new BadRequestException(
        'Amount mismatch',
      );
    }

    // Risky transactions must be reviewed.

    if (String(verified.risk_level) === '1') {
      await this.prisma.payment.updateMany({
        where: {
          id: payment.id,
          status: {
            in: ['PENDING', 'INITIATED'],
          },
        },
        data: {
          status: 'REVIEW_REQUIRED',
        },
      });

      return {
        success: false,
        message: 'Payment requires manual review',
      };
    }

    return this.confirmation.confirmPayment(
      payment.transactionId,
      {
        amount: verified.amount,
        currency: verified.currency,
        gatewayTransactionId: verified.val_id,
      },
    );
  }
}
```

এখানে IPN-এর `status` বা `verify_sign` একা বিশ্বাস করা হয়নি। Gateway-এর Validation API থেকে পাওয়া Transaction Details-ই মূল ভিত্তি।

SSLCommerz-এর IPN এবং Order Validation নির্দেশনা: SSLCOMMERZ +1

আরও একটি জরুরি বিষয়: `REVIEW_REQUIRED` Payment স্বয়ংক্রিয়ভাবে Paid হবে না। Admin Review সম্পন্ন না হওয়া পর্যন্ত Order Fulfillment বন্ধ থাকবে।

### ১০. Payment Controller

এখন সব API Endpoint এক জায়গায় যুক্ত করি।

`src/payment/payment.controller.ts`

```typescript

import {
  Controller,
  Post,
  Get,
  Body,
  Query,
  Req,
  UseGuards,
  Res,
} from '@nestjs/common';

import { Response } from 'express';

@Controller('payments')
export class PaymentController {
  constructor(
    private readonly bkashCallback: BkashCallbackService,
    private readonly sslWebhook: SSLCommerzWebhookService,
  ) {}

  @Get('bkash/callback')
  async bkashCallbackHandler(
    @Query('paymentID') paymentID: string,
    @Query('status') status: string,
    @Res() res: Response,
  ) {
    try {
      await this.bkashCallback.handleCallback(
        paymentID,
        status,
      );

      return res.redirect(
        `${process.env.FRONTEND_URL}/payment/success`,
      );

    } catch {
      return res.redirect(
        `${process.env.FRONTEND_URL}/payment/pending`,
      );
    }
  }

  @Post('sslcommerz/ipn')
  async sslIPN(@Body() body: any) {
    return this.sslWebhook.handleIPN(body);
  }

  @Post('sslcommerz/success')
  async sslSuccess(
    @Res() res: Response,
  ) {
    // Browser redirect is not payment proof.
    return res.redirect(
      `${process.env.FRONTEND_URL}/payment/pending`,
    );
  }

  @Post('sslcommerz/fail')
  async sslFail(
    @Res() res: Response,
  ) {
    return res.redirect(
      `${process.env.FRONTEND_URL}/payment/failed`,
    );
  }

  @Post('sslcommerz/cancel')
  async sslCancel(
    @Res() res: Response,
  ) {
    return res.redirect(
      `${process.env.FRONTEND_URL}/payment/cancelled`,
    );
  }
}
```

SSLCommerz IPN Route-এ Express-এর URL-encoded Form Body Parser চালু থাকতে হবে। NestJS-এর সাধারণ Express Adapter-এ এটি সাধারণত ডিফল্টভাবে কাজ করে। Raw JSON ধরে নেবেন না।

এখানে Browser-এর Success Redirect শুধু Customer Experience-এর জন্য। Payment Confirmation কেবল IPN Verification-এর পর হবে।

### ১১. Expired Voucher Reservation Release Cron

এখন ধরুন Customer Voucher Reserve করলেন, কিন্তু Payment সম্পন্ন করলেন না।

১৫ মিনিট পর Voucher আবার `ACTIVE` হবে।

`src/payment/payment-expiry.cron.ts`

```typescript

import {
  Injectable,
  Logger,
} from '@nestjs/common';

import {
  Cron,
  CronExpression,
} from '@nestjs/schedule';

import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class PaymentExpiryCron {
  private readonly logger = new Logger(
    PaymentExpiryCron.name,
  );

  constructor(
    private readonly prisma: PrismaService,
  ) {}

  @Cron(CronExpression.EVERY_MINUTE)
  async releaseExpiredReservations() {

    const now = new Date();

    const expired =
      await this.prisma.voucher.findMany({
        where: {
          status: 'RESERVED',
          reservationExpiresAt: {
            lte: now,
          },
        },
        select: {
          id: true,
          reservedOrderId: true,
        },
      });

    for (const item of expired) {
      try {
        await this.prisma.$transaction(
          async (tx) => {

            const result =
              await tx.voucher.updateMany({
                where: {
                  id: item.id,
                  status: 'RESERVED',
                  reservationExpiresAt: {
                    lte: now,
                  },
                },
                data: {
                  status: 'ACTIVE',
                  reservedOrderId: null,
                  reservationExpiresAt: null,
                },
              });

            if (result.count !== 1) {
              return;
            }

            // Mark unfinished payment attempts expired.

            if (item.reservedOrderId) {
              await tx.payment.updateMany({
                where: {
                  orderId: item.reservedOrderId,
                  status: {
                    in: ['INITIATED', 'PENDING'],
                  },
                },
                data: {
                  status: 'EXPIRED',
                },
              });
            }
          },
        );

      } catch (error) {
        this.logger.error(
          `Reservation release failed: ${item.id}`,
          error instanceof Error
            ? error.stack
            : String(error),
        );
      }
    }
  }
}
```

Production সতর্কতা

শুধু ১৫ মিনিট পার হলেই Gateway Payment নিশ্চিতভাবে ব্যর্থ হয়েছে ধরে নেওয়া যাবে না। Customer ঠিক শেষ মুহূর্তে Payment সম্পন্ন করতে পারেন, অথচ IPN পরে আসতে পারে।

তাই Production-এ Expiry Job-এর আগে Gateway Query API দিয়ে Payment Status যাচাই করুন। Payment সফল হলে Voucher Release না করে Confirmation করুন। Payment Pending বা Unknown থাকলে Reservation বাড়িয়ে দিন অথবা Manual Review-তে পাঠান।

### ১২. Database-এ Gateway Transaction ID সংরক্ষণ

আগের Payment Model-এ এই Field যোগ করুন:

```prisma
model Payment {
  // Existing fields ...

  gatewayTransactionId String? @unique
}
```

এবং `PaymentConfirmationService`-এ Payment Update করার সময়:

```typescript
data: {
  status: 'PAID',
  paidAt: new Date(),
  gatewayTransactionId:
    verified.gatewayTransactionId,
},
```

এতে bKash-এর `trxID` এবং SSLCommerz-এর `val_id` সংরক্ষিত থাকবে।

### ১৩. Next.js Checkout Integration

Frontend থেকে শুধু Payment Gateway নির্বাচন এবং Order ID পাঠাবেন।

`app/checkout/payment/page.tsx`

```tsx

'use client';

import { useState } from 'react';

export default function PaymentPage() {
  const [gateway, setGateway] =
    useState('BKASH');

  const [loading, setLoading] =
    useState(false);

  const [error, setError] =
    useState('');

  async function handlePayment() {
    setLoading(true);
    setError('');

    try {
      const response = await fetch(
        `${process.env.NEXT_PUBLIC_API_URL}/payments/initiate`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          credentials: 'include',
          body: JSON.stringify({
            orderId: 'YOUR_ORDER_ID',
            provider: gateway,
          }),
        },
      );

      const result = await response.json();

      if (!response.ok) {
        throw new Error(
          result.message || 'Payment initiation failed',
        );
      }

      window.location.assign(
        result.redirectUrl,
      );

    } catch (err) {
      setError(
        err instanceof Error
          ? err.message
          : 'Something went wrong',
      );
    } finally {
      setLoading(false);
    }
  }

  return (
    <div className="max-w-lg mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">
        Select Payment Method
      </h1>

      <div className="space-y-3">

        <label className="block border rounded-lg p-4">
          <input
            type="radio"
            value="BKASH"
            checked={gateway === 'BKASH'}
            onChange={() => setGateway('BKASH')}
          />
          <span className="ml-3">
            bKash
          </span>
        </label>

        <label className="block border rounded-lg p-4">
          <input
            type="radio"
            value="SSLCOMMERZ"
            checked={gateway === 'SSLCOMMERZ'}
            onChange={() =>
              setGateway('SSLCOMMERZ')
            }
          />
          <span className="ml-3">
            SSLCommerz
          </span>
        </label>

      </div>

      {error && (
        <p className="text-red-600 mt-4">
          {error}
        </p>
      )}

      <button
        onClick={handlePayment}
        disabled={loading}
        className="w-full bg-blue-600 text-white p-3 rounded-lg mt-6"
      >
        {loading ? 'Processing...' : 'Pay Now'}
      </button>
    </div>
  );
}
```

`YOUR_ORDER_ID` অবশ্যই আপনার authenticated Checkout Context থেকে আসবে। API-তে Customer Identity JWT বা Session থেকে নিতে হবে।

### ১৪. NestJS Module Registration

`payment.module.ts`

```typescript
@Module({
  imports: [
    PrismaModule,
    ConfigModule,
    ScheduleModule.forRoot(),
  ],
  controllers: [
    PaymentController,
  ],
  providers: [
    PaymentService,
    BkashService,
    SSLCommerzService,
    BkashCallbackService,
    SSLCommerzWebhookService,
    PaymentConfirmationService,
    PaymentExpiryCron,
  ],
})
export class PaymentModule {}
```

`ScheduleModule.forRoot()` শুধু Root App Module-এ একবার Register করবেন। Payment Module-এ পুনরায় Register করার প্রয়োজন নেই।

### ১৫. Payment Testing Checklist

Production-এ যাওয়ার আগে Sandbox Environment-এ নিচের Scenario-গুলো পরীক্ষা করুন।

| Test Scenario                 | Expected Result                  |
| ----------------------------- | -------------------------------- |
| bKash Successful Payment      | Order PAID                       |
| SSLCommerz Successful Payment | Order PAID                       |
| Payment Cancelled             | Order Paid নয়                    |
| Insufficient Payment          | Order Paid নয়                    |
| Amount Mismatch               | Reject                           |
| Duplicate IPN                 | Duplicate Redemption নয়          |
| Duplicate Callback            | Duplicate Confirmation নয়        |
| Expired Voucher               | Redemption Reject                |
| Voucher Reserved              | অন্য Order ব্যবহার করতে পারবে না |
| Risky SSL Payment             | Manual Review                    |
| Gateway Timeout               | Payment Status Reconciliation    |

### ১৬. Production Security Checklist

Gateway Credentials Backend Environment-এ রাখা

HTTPS ও TLS 1.2+ ব্যবহার করা

Gateway Response Server-side Verify করা

Payment Amount Database থেকে নেওয়া

Webhook Idempotency Implement করা

Database Transaction ও Unique Constraints ব্যবহার করা

Payment Timeout Reconciliation চালু করা

Duplicate Payment-এর Refund Workflow রাখা

Gateway Logs-এ Secret ও Sensitive Data না রাখা

Payment Success-এর আগে Order Fulfillment বন্ধ রাখা

আরও তিনটি Production Improvement প্রয়োজন:

* Payment Retry: Network Timeout হলে নতুন Payment না বানিয়ে আগের Transaction Query করতে হবে।

* Refund Management: একই Order-এর জন্য দুই Gateway-তে সফল Payment হয়ে গেলে অতিরিক্ত Payment Refund করতে হবে।

* Durable Webhook Processing: IPN গ্রহণের পর Job Queue-তে Event সংরক্ষণ করে Processing করতে হবে, যাতে Server Restart হলেও Event হারিয়ে না যায়।

### ১৭. Official Documentation

এগুলো আপনার Implementation-এর Reference হিসেবে রাখুন।

* [bKash Developer Documentation](https://developer.bka.sh/docs?utm_source=chatgpt.com)

* [SSLCommerz Developer Documentation](https://developer.sslcommerz.com/doc/v4/?utm_source=chatgpt.com)

* [Prisma Transactions Documentation](https://www.prisma.io/docs/orm/prisma-client/queries/transactions?utm_source=chatgpt.com)

### ১৮. আপনার E-commerce System-এর বর্তমান Architecture

### Loyalty & Payment Platform

* Loyalty Points Management

    Implemented

* Promotional Campaign & Pool

    Implemented

* Automatic Winner Selection

    Implemented

* Voucher Generator

    Implemented

* Customer Voucher Wallet

    Implemented

* bKash + SSLCommerz Integration

    Implementation Provided

* Payment Reconciliation & Refund

    Next Stage

* Admin Analytics & Monitoring

    Next Stage

### আমার পরবর্তী সুপারিশ

আপনার সিস্টেমের জন্য পরবর্তী ধাপে Production-grade Payment Reconciliation, Refund Management & BullMQ + Redis Webhook Queue তৈরি করা উচিত।

বিশেষ করে bKash ও SSLCommerz-এর ক্ষেত্রে Payment সফল হলেও Network Failure-এর কারণে আপনার Database আপডেট না হওয়ার পরিস্থিতি সামলাতে এটি খুবই গুরুত্বপূর্ণ।

এই তিনটি Module যুক্ত হলে আপনার Payment System অনেক বেশি নির্ভরযোগ্য হবে।
