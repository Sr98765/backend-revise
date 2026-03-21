# backend-revise

=========================================================================
nvm install 22
nvm use 22
node -v
==========================================
mkdir backend
cd backend

npm install -g @nestjs/cli
nest --version
========================================================

sudo apt update
sudo apt install postgresql postgresql-contrib -y

sudo service postgresql start

sudo service postgresql status

sudo su postgres
psql

CREATE DATABASE viral_auth_db;
CREATE DATABASE viral_wallet_db;

CREATE USER viral_user WITH PASSWORD 'viral123';

GRANT ALL PRIVILEGES ON DATABASE viral_auth_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_wallet_db TO viral_user;

\c viral_auth_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_wallet_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\q
exit

psql -h localhost -U viral_user -d viral_auth_db
viral123   [password]


sudo su postgres
psql
ALTER USER viral_user CREATEDB;
====================================================================================================================
                             auth-service
=====================================================================================================================  



nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new auth-service             [npm]
cd auth-service
==================
[Install Auth dependencies]
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
=======================================================================
prisma
---------
npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init
=============================

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_auth_db"            [.env]

========================
[prisma/schema.prisma]
--------------------------
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
}

====================================================
npx prisma generate

npx prisma migrate dev --name init

npx prisma studio
========================================

npx nest g service prisma
--------------------
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect()
  }

  async onModuleDestroy() {
    await this.$disconnect()
  }
}
============================
npx nest g module auth
npx nest g service auth
npx nest g controller auth
-----------------------
[src/auth/auth.module.ts]

import { Module } from '@nestjs/common'
import { JwtModule } from '@nestjs/jwt'
import { AuthService } from './auth.service'
import { AuthController } from './auth.controller'
import { PrismaService } from '../prisma/prisma.service'

@Module({
  imports: [
    JwtModule.register({
      secret: 'viral_secret',
      signOptions: { expiresIn: '1h' },
    }),
  ],
  providers: [AuthService, PrismaService],
  controllers: [AuthController],
})
export class AuthModule {}

===================================
[src/auth/auth.service.ts]

import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { JwtService } from '@nestjs/jwt'
import * as bcrypt from 'bcrypt'

@Injectable()
export class AuthService {

  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string) {

    const hashed = await bcrypt.hash(password, 10)

    const user = await this.prisma.user.create({
      data: {
        email,
        password: hashed,
      },
    })

    return { id: user.id, email: user.email }
  }

  async login(email: string, password: string) {

    const user = await this.prisma.user.findUnique({
      where: { email },
    })

    if (!user) throw new Error('User not found')

    const valid = await bcrypt.compare(password, user.password)

    if (!valid) throw new Error('Invalid password')

    const token = this.jwtService.sign({
      id: user.id,
      email: user.email,
    })

    return { token }
  }
}
==============================
[src/auth/auth.controller.ts]

import { Controller, Post, Body } from '@nestjs/common'
import { AuthService } from './auth.service'

@Controller('auth')
export class AuthController {

  constructor(private authService: AuthService) {}

  @Post('register')
  register(@Body() body: { email: string; password: string }) {
    return this.authService.register(body.email, body.password)
  }

  @Post('login')
  login(@Body() body: { email: string; password: string }) {
    return this.authService.login(body.email, body.password)
  }

}
======================================

[src/main.ts]

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3001)
}
bootstrap();


=============================
npx prisma generate
npx prisma migrate dev --name init

npm run start:dev
npx prisma studio
==========================
[Register]

curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Login]

curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

===========================================================================================================================================
====================================================================================================================
                             Wallet-service
=====================================================================================================================  



nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new wallet-service             [npm]
cd wallet-service
==========================

npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt

npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init

======================================

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_wallet_db"        [.env]

==========================================
 [prisma/schema.prisma]
 
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Wallet {
  id           Int           @id @default(autoincrement())
  userId       Int           @unique
  balance      Float         @default(0)
  transactions Transaction[]
}

model Transaction {
  id        Int      @id @default(autoincrement())
  walletId  Int
  amount    Float
  type      String
  createdAt DateTime @default(now())

  wallet    Wallet   @relation(fields: [walletId], references: [id])
}

==============================================================================================
npx prisma generate

npx prisma migrate dev --name init
===============================================

npx nest g service prisma
-------------------------------
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect()
  }

  async onModuleDestroy() {
    await this.$disconnect()
  }
}
=====================================================================

npx nest g module wallet
npx nest g service wallet
npx nest g controller wallet
--------------------------------------
[src/wallet/wallet.module.ts]

import { Module } from '@nestjs/common'
import { JwtModule } from '@nestjs/jwt'
import { WalletService } from './wallet.service'
import { WalletController } from './wallet.controller'
import { PrismaService } from '../prisma/prisma.service'

@Module({
  imports: [
    JwtModule.register({
      secret: 'viral_secret',
      signOptions: { expiresIn: '1h' },
    }),
  ],
  providers: [WalletService, PrismaService],
  controllers: [WalletController],
})
export class WalletModule {}

========================================================================

[src/wallet/wallet.service.ts]

import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class WalletService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  /**
   * Get wallet balance for a user.
   * If wallet doesn't exist, it will be automatically created with balance 0.
   */
  async getBalance(userId: number) {
    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: {},
      create: { userId, balance: 0 },
    });

    return { balance: wallet.balance };
  }

  /**
   * Deposit money into user's wallet.
   * Creates the wallet automatically if it doesn't exist.
   */
  async deposit(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Deposit amount must be greater than 0');

    // Upsert wallet (create if not exists)
    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: { balance: { increment: amount } },
      create: { userId, balance: amount },
    });

    // Record the deposit transaction
    await this.prisma.transaction.create({
      data: {
        walletId: wallet.id,
        type: 'deposit',
        amount,
      },
    });

    return { balance: wallet.balance };
  }

  /**
   * Withdraw money from user's wallet.
   * Throws error if balance is insufficient.
   */
  async withdraw(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Withdraw amount must be greater than 0');

    // Find the wallet
    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) throw new BadRequestException('Wallet not found');
    if (wallet.balance < amount) throw new BadRequestException('Insufficient balance');

    // Update balance
    const updated = await this.prisma.wallet.update({
      where: { userId },
      data: { balance: { decrement: amount } },
    });

    // Record the withdrawal transaction
    await this.prisma.transaction.create({
      data: {
        walletId: wallet.id,
        type: 'withdraw',
        amount,
      },
    });

    return { balance: updated.balance };
  }

  /**
   * Optional: Get full transaction history for a user
   */
  async getTransactions(userId: number) {
    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) return [];

    const transactions = await this.prisma.transaction.findMany({
      where: { walletId: wallet.id },
      orderBy: { createdAt: 'desc' },
    });

    return transactions;
  }
}
=========================================================================

[src/wallet/wallet.controller.ts]

import { Controller, Get, Post, Body, Headers, BadRequestException } from '@nestjs/common';
import { WalletService } from './wallet.service';
import { JwtService } from '@nestjs/jwt';

@Controller('wallet')
export class WalletController {
  constructor(
    private walletService: WalletService,
    private jwtService: JwtService,
  ) {}

  /**
   * Helper: Extract userId from Authorization header (Bearer token)
   */
  private getUserId(authHeader: string): number {
    if (!authHeader) throw new BadRequestException('No Authorization header provided');
    try {
      const token = authHeader.split(' ')[1]; // Bearer <token>
      const decoded: any = this.jwtService.verify(token, { secret: 'viral_secret' });
      return decoded.id;
    } catch (error) {
      throw new BadRequestException('Invalid or expired token');
    }
  }

  /**
   * GET /wallet/balance
   * Returns the current balance for the logged-in user
   */
  @Get('balance')
  async getBalance(@Headers('authorization') auth: string) {
    const userId = this.getUserId(auth);
    return this.walletService.getBalance(userId);
  }

  /**
   * POST /wallet/deposit
   * Body: { amount: number }
   */
  @Post('deposit')
  async deposit(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = this.getUserId(auth);
    if (!body.amount) throw new BadRequestException('Amount is required');
    return this.walletService.deposit(userId, body.amount);
  }

  /**
   * POST /wallet/withdraw
   * Body: { amount: number }
   */
  @Post('withdraw')
  async withdraw(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = this.getUserId(auth);
    if (!body.amount) throw new BadRequestException('Amount is required');
    return this.walletService.withdraw(userId, body.amount);
  }

  /**
   * GET /wallet/transactions
   * Returns full transaction history for the logged-in user
   */
  @Get('transactions')
  async getTransactions(@Headers('authorization') auth: string) {
    const userId = this.getUserId(auth);
    return this.walletService.getTransactions(userId);
  }
}

==================================================================================

[src/main.ts]

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3002)
}
bootstrap();

==========================================

npx prisma generate
npx prisma migrate dev --name init

npm run start:dev
npx prisma studio

========================================
[Register]
curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Login]
curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"test@viral.com","password":"123456"}'

[Balance]
curl -X GET http://localhost:3002/wallet/balance \
-H "Authorization: Bearer JWT_TOKEN"

[Deposit]
curl -X POST http://localhost:3002/wallet/deposit \
-H "Authorization: Bearer JWT_TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":100}'

[Withdraw]
curl -X POST http://localhost:3002/wallet/withdraw \
-H "Authorization: Bearer JWT_TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":50}'

[Transactions]
curl -X GET http://localhost:3002/wallet/transactions \
-H "Authorization: Bearer JWT_TOKEN"

================================================================================================================
================================================================================================================
                                  Game-service setup
=============================================================================================================

nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new game-service             [npm]
cd game-service
=====================================================================
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt


npm install prisma@4 --save-dev
npm install @prisma/client@4
================================================================
sudo su postgres
psql

CREATE DATABASE viral_game_db;
GRANT ALL PRIVILEGES ON DATABASE viral_game_db TO viral_user;

\c viral_game_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\q
exit

npx prisma init
======================================================

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_game_db"            [.env]

==========================================================
prisma/schema.prisma
-----------------------

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Player {
  id       Int         @id @default(autoincrement())
  username String      @unique
  rounds   GameRound[]
}

model GameRound {
  id        Int      @id @default(autoincrement())
  playerId  Int
  bet       Float
  result    String
  payout    Float
  createdAt DateTime @default(now())

  player    Player   @relation(fields: [playerId], references: [id])
}


============================================================================

npx prisma generate
npx prisma migrate dev --name init
npx prisma studio

==================================================================

npx nest g service prisma
==================================
[src/prisma/prisma.service.ts]

import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
=====================================================

npx nest g module game
npx nest g service game
npx nest g controller game

========================================
[src/game/game.module.ts]

import { Module } from '@nestjs/common';
import { GameService } from './game.service';
import { GameController } from './game.controller';
import { PrismaService } from '../prisma/prisma.service';

@Module({
  imports: [],
  providers: [GameService, PrismaService],
  controllers: [GameController],
})
export class GameModule {}

==========================================
[src/game.service.ts]

import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class GameService {
  constructor(private prisma: PrismaService) {}

  // Create a new player
  async createPlayer(username: string) {
  return this.prisma.player.create({
    data: { username },
  });
}

  // Get all players
  async getPlayers() {
  return this.prisma.player.findMany();
}

  // Play a game round
  async playRound(playerId: number, bet: number) {
    if (bet <= 0) throw new BadRequestException('Bet must be > 0');

    // 50/50 simple game logic
    const win = Math.random() < 0.5;
    const payout = win ? bet * 2 : 0;
    const result = win ? 'win' : 'lose';

    // Record round in DB
    return this.prisma.gameRound.create({
      data: { playerId, bet, result, payout },
    });
  }

  // Get all rounds for a player
  async getRounds(playerId: number) {
    return this.prisma.gameRound.findMany({
      where: { playerId },
      orderBy: { createdAt: 'desc' },
    });
  }
}

==================================================================
[src/game.controller.ts]

import { Controller, Post, Body, Get, Param } from '@nestjs/common';
import { GameService } from './game.service';

@Controller('game')
export class GameController {
  constructor(private gameService: GameService) {}

  @Post('player')
  createPlayer(@Body() body: { username: string }) {
    return this.gameService.createPlayer(body.username);
  }

  @Get('players')
  getPlayers() {
    return this.gameService.getPlayers();
  }

  @Post('play')
  playRound(@Body() body: { playerId: number; bet: number }) {
    return this.gameService.playRound(body.playerId, body.bet);
  }

  @Get('rounds/:playerId')
  getRounds(@Param('playerId') playerId: string) {
    return this.gameService.getRounds(Number(playerId));
  }
}
=========================================================
[main.ts]


import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  await app.listen(3003); // Game-Service port
}
bootstrap();

==============================================================

npm run start:dev
=============================
[Create player]

curl -X POST http://localhost:3003/game/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[List players]
curl -X GET http://localhost:3003/game/players

[Play round]
curl -X POST http://localhost:3003/game/play \
-H "Content-Type: application/json" \
-d '{"playerId":1,"bet":100}'

[Get player Rounds]
curl -X GET http://localhost:3003/game/rounds/1

============================================================================================================
=========================================================================================================
                                 History-service
======================================================================================================
 History Microservice (with Prisma + PostgreSQL)

---

## **1️⃣ Setup Node & Nest**

```bash
nvm use 22
node -v
```

```bash
cd /workspaces/backend-revise/backend
nest new history-service
cd history-service
```

---

## **2️⃣ Install Dependencies**

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
npm install prisma@4 --save-dev
npm install @prisma/client@4
```

---

## **3️⃣ Initialize Prisma**

```bash
npx prisma init
```

Edit `.env`:

```env
DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_history_db"
```

> Make sure you created the database beforehand:

```sql
CREATE DATABASE viral_history_db;
GRANT ALL PRIVILEGES ON DATABASE viral_history_db TO viral_user;
\c viral_history_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;
```

---

## **4️⃣ Define Prisma Schema**

Edit `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model PlayerStats {
  id          Int      @id @default(autoincrement())
  username    String   @unique
  totalBets   Float    @default(0)
  totalWins   Float    @default(0)
  totalLosses Float    @default(0)
  createdAt   DateTime @default(now())
}
```

---

## **5️⃣ Prisma Generate & Migrate**

```bash
npx prisma generate
npx prisma migrate dev --name init
npx prisma studio  # optional GUI to check DB
```

---

## **6️⃣ Generate Prisma Service in Nest**

```bash
npx nest g service prisma
```

`src/prisma/prisma.service.ts`:

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient
  implements OnModuleInit, OnModuleDestroy {

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

---

## **7️⃣ Generate History Module, Service, Controller**

```bash
npx nest g module history
npx nest g service history
npx nest g controller history
```

---

## **8️⃣ Implement History Service**

`src/history/history.service.ts`:

```ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class HistoryService {
  constructor(private prisma: PrismaService) {}

  async getPlayerStats(username: string) {
    if (!username) throw new BadRequestException('Username is required');

    const player = await this.prisma.playerStats.findUnique({
      where: { username },
    });

    return player || null;
  }

  async upsertPlayerStats(username: string, bets: number, wins: number, losses: number) {
    if (!username) throw new BadRequestException('Username is required');

    return this.prisma.playerStats.upsert({
      where: { username },
      create: { username, totalBets: bets, totalWins: wins, totalLosses: losses },
      update: {
        totalBets: { increment: bets },
        totalWins: { increment: wins },
        totalLosses: { increment: losses },
      },
    });
  }

  async getLeaderboard(limit = 10) {
    return this.prisma.playerStats.findMany({
      orderBy: { totalWins: 'desc' },
      take: limit,
    });
  }
}
```

---

## **9️⃣ Implement History Controller**

`src/history/history.controller.ts`:

```ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { HistoryService } from './history.service';

@Controller('history')
export class HistoryController {
  constructor(private historyService: HistoryService) {}

  @Get('leaderboard')
  async leaderboard() {
    return this.historyService.getLeaderboard();
  }

  @Post('upsert')
  async upsert(@Body() body: { username: string; bets: number; wins: number; losses: number }) {
    const { username, bets, wins, losses } = body;
    return this.historyService.upsertPlayerStats(username, bets, wins, losses);
  }

  @Get('player')
  async getPlayer(@Body() body: { username: string }) {
    return this.historyService.getPlayerStats(body.username);
  }
}
```

---

## **🔟 Update History Module**

`src/history/history.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { HistoryService } from './history.service';
import { HistoryController } from './history.controller';
import { PrismaService } from '../prisma/prisma.service';

@Module({
  providers: [HistoryService, PrismaService],
  controllers: [HistoryController],
})
export class HistoryModule {}
```

---

## **1️⃣1️⃣ Update App Module**

`src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { HistoryModule } from './history/history.module';
import { PrismaService } from './prisma/prisma.service';

@Module({
  imports: [HistoryModule],
  providers: [PrismaService],
})
export class AppModule {}
```

---

## **1️⃣2️⃣ Update main.ts**

`src/main.ts`:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  await app.listen(3003); // history-service port
}
bootstrap();
```

---

## **1️⃣3️⃣ Start Service**

```bash
npm run start:dev
```

---

## **1️⃣4️⃣ Test APIs with cURL**

**Upsert stats:**

```bash
curl -X POST http://localhost:3003/history/upsert \
-H "Content-Type: application/json" \
-d '{"username":"player1","bets":5,"wins":3,"losses":2}'
```

**Get player stats:**

```bash
curl -X GET http://localhost:3003/history/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'
```

**Get leaderboard:**

```bash
curl -X GET http://localhost:3003/history/leaderboard
```

