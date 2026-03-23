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
CREATE DATABASE viral_game_db;
CREATE DATABASE viral_history_db;

CREATE USER viral_user WITH PASSWORD 'viral123';

GRANT ALL PRIVILEGES ON DATABASE viral_auth_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_wallet_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_game_db TO viral_user;
GRANT ALL PRIVILEGES ON DATABASE viral_history_db TO viral_user;

\c viral_auth_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_wallet_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_game_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\c viral_history_db
GRANT ALL PRIVILEGES ON SCHEMA public TO viral_user;

\q
exit

psql -h localhost -U viral_user -d viral_auth_db
viral123   [password]


sudo su postgres
psql
ALTER USER viral_user CREATEDB;

===============================================================================================
===============================================================================================

nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
nest new auth-service
nest new wallet-service  
nest new game-service  
nest new history-service

[npm]
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

import { Injectable, UnauthorizedException } from '@nestjs/common'
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
      data: { email, password: hashed },
    })
    return { id: user.id, email: user.email }
  }

  async login(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } })
    if (!user) throw new Error('User not found')
    const valid = await bcrypt.compare(password, user.password)
    if (!valid) throw new Error('Invalid password')
    const token = this.jwtService.sign({ id: user.id, email: user.email })
    return { token }
  }

  // ✅ NEW — called by wallet-service
  async verifyToken(token: string) {
    try {
      const payload = this.jwtService.verify(token)
      return { id: payload.id, email: payload.email }
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }
}

=========================================================
[src/auth/auth.controller.ts]

import { Controller, Post, Body, Headers, UnauthorizedException } from '@nestjs/common'
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

  // ✅ NEW — other services POST here to validate a Bearer token
  @Post('verify')
  verify(@Headers('authorization') auth: string) {
    if (!auth) throw new UnauthorizedException('No token provided')
    const token = auth.split(' ')[1]
    return this.authService.verifyToken(token)
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

[verify]

curl -X POST http://localhost:3001/auth/verify \
-H "Authorization: Bearer YOUR_JWT_TOKEN"
===========================================================================================================================================
====================================================================================================================
                             Wallet-service
=====================================================================================================================  
cd -


nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
cd wallet-service
==========================

npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
npm install axios

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
import { WalletService } from './wallet.service'
import { WalletController } from './wallet.controller'
import { PrismaService } from '../prisma/prisma.service'

@Module({
  imports: [],                          // ✅ JwtModule removed
  providers: [WalletService, PrismaService],
  controllers: [WalletController],
})
export class WalletModule {}

========================================================================

[src/wallet/wallet.service.ts]

import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class WalletService {
  constructor(
    private prisma: PrismaService,
    // ✅ JwtService removed — wallet no longer handles tokens
  ) {}

  async getBalance(userId: number) {
    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: {},
      create: { userId, balance: 0 },
    });
    return { balance: wallet.balance };
  }

  async deposit(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Deposit amount must be greater than 0');

    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: { balance: { increment: amount } },
      create: { userId, balance: amount },
    });

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'deposit', amount },
    });

    return { balance: wallet.balance };
  }

  async withdraw(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Withdraw amount must be greater than 0');

    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) throw new BadRequestException('Wallet not found');
    if (wallet.balance < amount) throw new BadRequestException('Insufficient balance');

    const updated = await this.prisma.wallet.update({
      where: { userId },
      data: { balance: { decrement: amount } },
    });

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'withdraw', amount },
    });

    return { balance: updated.balance };
  }

  async getTransactions(userId: number) {
    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) return [];

    return this.prisma.transaction.findMany({
      where: { walletId: wallet.id },
      orderBy: { createdAt: 'desc' },
    });
  }
}
=========================================================================

[src/wallet/wallet.controller.ts]

import { Controller, Get, Post, Body, Headers, BadRequestException, UnauthorizedException } from '@nestjs/common'
import { WalletService } from './wallet.service'
import axios from 'axios'

const AUTH_SERVICE_URL = 'http://localhost:3001'

@Controller('wallet')
export class WalletController {
  constructor(private walletService: WalletService) {}

  // ✅ Now delegates token verification to auth-service
  private async getUserId(authHeader: string): Promise<number> {
    if (!authHeader) throw new BadRequestException('No Authorization header provided')
    try {
      const response = await axios.post(
        `${AUTH_SERVICE_URL}/auth/verify`,
        {},
        { headers: { authorization: authHeader } },
      )
      return response.data.id
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }

  @Get('balance')
  async getBalance(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getBalance(userId)
  }

  @Post('deposit')
  async deposit(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = await this.getUserId(auth)
    if (!body.amount) throw new BadRequestException('Amount is required')
    return this.walletService.deposit(userId, body.amount)
  }

  @Post('withdraw')
  async withdraw(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = await this.getUserId(auth)
    if (!body.amount) throw new BadRequestException('Amount is required')
    return this.walletService.withdraw(userId, body.amount)
  }

  @Get('transactions')
  async getTransactions(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getTransactions(userId)
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
cd -

nvm use 22
node -v
==================
cd /workspaces/backend-revise/backend
ls
==================
cd game-service
=====================================================================
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
npm install axios

npm install prisma@4 --save-dev
npm install @prisma/client@4

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

import { Injectable, BadRequestException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import axios from 'axios'

const WALLET_SERVICE_URL = 'http://localhost:3002'
const HISTORY_SERVICE_URL = 'http://localhost:3004'

@Injectable()
export class GameService {
  constructor(private prisma: PrismaService) {}

  async createPlayer(username: string) {
    return this.prisma.player.create({
      data: { username },
    })
  }

  async getPlayers() {
    return this.prisma.player.findMany()
  }

  // ✅ Now wired to wallet + history
  async playRound(playerId: number, bet: number, authHeader: string) {
    if (bet <= 0) throw new BadRequestException('Bet must be > 0')

    // 1️⃣ Deduct bet from wallet (fails if insufficient balance)
    try {
      await axios.post(
        `${WALLET_SERVICE_URL}/wallet/withdraw`,
        { amount: bet },
        { headers: { authorization: authHeader } },
      )
    } catch (err) {
      throw new BadRequestException(
        err.response?.data?.message || 'Wallet withdraw failed'
      )
    }

    // 2️⃣ Run game logic
    const win = Math.random() < 0.5
    const payout = win ? bet * 2 : 0
    const result = win ? 'win' : 'lose'

    // 3️⃣ Pay out winnings if player won
    if (win) {
      await axios.post(
        `${WALLET_SERVICE_URL}/wallet/deposit`,
        { amount: payout },
        { headers: { authorization: authHeader } },
      )
    }

    // 4️⃣ Save round in game DB
    const round = await this.prisma.gameRound.create({
      data: { playerId, bet, result, payout },
    })

    // 5️⃣ Push stats to history-service (fire-and-forget, don't block)
    const player = await this.prisma.player.findUnique({ where: { id: playerId } })
    if (player) {
      axios.post(`${HISTORY_SERVICE_URL}/history/upsert`, {
        username: player.username,
        bets: bet,
        wins: win ? payout : 0,
        losses: win ? 0 : bet,
      }).catch(() => {})  // silent fail — history is non-critical
    }

    return round
  }

  async getRounds(playerId: number) {
    return this.prisma.gameRound.findMany({
      where: { playerId },
      orderBy: { createdAt: 'desc' },
    })
  }
}

==================================================================
[src/game.controller.ts]

import { Controller, Post, Body, Get, Param, Headers, BadRequestException } from '@nestjs/common'
import { GameService } from './game.service'

@Controller('game')
export class GameController {
  constructor(private gameService: GameService) {}

  @Post('player')
  createPlayer(@Body() body: { username: string }) {
    return this.gameService.createPlayer(body.username)
  }

  @Get('players')
  getPlayers() {
    return this.gameService.getPlayers()
  }

  // ✅ Now requires Authorization header
  @Post('play')
  playRound(
    @Headers('authorization') auth: string,
    @Body() body: { playerId: number; bet: number },
  ) {
    if (!auth) throw new BadRequestException('Authorization header required')
    return this.gameService.playRound(body.playerId, body.bet, auth)
  }

  @Get('rounds/:playerId')
  getRounds(@Param('playerId') playerId: string) {
    return this.gameService.getRounds(Number(playerId))
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
cd -

nvm use 22
node -v
=====================================

cd /workspaces/backend-revise/backend

cd history-service

==========================

npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt

npm install prisma@4 --save-dev
npm install @prisma/client@4

npx prisma init

==========================================

DATABASE_URL="postgresql://viral_user:viral123@localhost:5432/viral_history_db"     [.env]

===========================================
[ schema.prisma]


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

========================================================

npx prisma generate
npx prisma migrate dev --name init

npx prisma studio  

==========================================

npx nest g service prisma

===============================

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

======================================================

npx nest g module history
npx nest g service history
npx nest g controller history

=================================================

[src/history/history.service.ts]


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

===============================================

[src/history/history.controller.ts]


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

=======================================

[src/history/history.module.ts]


import { Module } from '@nestjs/common';
import { HistoryService } from './history.service';
import { HistoryController } from './history.controller';
import { PrismaService } from '../prisma/prisma.service';

@Module({
  providers: [HistoryService, PrismaService],
  controllers: [HistoryController],
})
export class HistoryModule {}

===============================================
[src/main.ts]

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  await app.listen(3004); // history-service port
}
bootstrap();

======================================

npm run start:dev

========================================

[Upsert stats]

curl -X POST http://localhost:3004/history/upsert \
-H "Content-Type: application/json" \
-d '{"username":"player1","bets":5,"wins":3,"losses":2}'

[Get player stats]

curl -X GET http://localhost:3004/history/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[Get leaderboard]

curl -X GET http://localhost:3004/history/leaderboard

==============================================================================================================================
==============================================================================================================================


Step 6 — Start all 4 services and test the full flow

Open 4 terminals:

[cd backend/auth-service]
npm run start:dev

[cd wallet-service]
npm run start:dev

[cd game-service]
npm run start:dev

[cd history-service] 
npm run start:dev
=====================================
Then run the full flow:

[Register]

curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Login]
curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Set token as variable (replace with your actual token)]
TOKEN="eyJhbGci..."

[Deposit funds]
curl -X POST http://localhost:3002/wallet/deposit \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":500}'

[Create a game player]
curl -X POST http://localhost:3003/game/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[Play a round (deducts bet, pays out if win, updates history)]
curl -X POST http://localhost:3003/game/play \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"playerId":1,"bet":100}'

[Check wallet balance — should reflect the result]
curl -X GET http://localhost:3002/wallet/balance \
-H "Authorization: Bearer $TOKEN"

[Check leaderboard]
curl -X GET http://localhost:3004/history/leaderboard
===================================================================================================================
























                                  Connection
==============================================================================================================================


Now let's connect them one service at a time.
=======================================================
Step 1 — Install axios in every service that makes outbound calls
Run these in their respective folders:
bash# wallet-service
cd /workspaces/backend-revise/backend/wallet-service

npm install axios

==========================================================
# game-service
cd /workspaces/backend-revise/backend/game-service

npm install axios
========================================================

auth-service and history-service don't call anyone — no axios needed there.

Step 2 — Auth service: expose a /auth/verify endpoint
The wallet-service needs to ask auth "is this token valid, and who is it?". Add one new endpoint.
[auth-service/src/auth/auth.service.ts] — add the verifyToken method:

import { Injectable, UnauthorizedException } from '@nestjs/common'
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
      data: { email, password: hashed },
    })
    return { id: user.id, email: user.email }
  }

  async login(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } })
    if (!user) throw new Error('User not found')
    const valid = await bcrypt.compare(password, user.password)
    if (!valid) throw new Error('Invalid password')
    const token = this.jwtService.sign({ id: user.id, email: user.email })
    return { token }
  }

  // ✅ NEW — called by wallet-service
  async verifyToken(token: string) {
    try {
      const payload = this.jwtService.verify(token)
      return { id: payload.id, email: payload.email }
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }
}

==========================================================================================
[auth-service/src/auth/auth.controller.ts] — add the /auth/verify route:

import { Controller, Post, Body, Headers, UnauthorizedException } from '@nestjs/common'
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

  // ✅ NEW — other services POST here to validate a Bearer token
  @Post('verify')
  verify(@Headers('authorization') auth: string) {
    if (!auth) throw new UnauthorizedException('No token provided')
    const token = auth.split(' ')[1]
    return this.authService.verifyToken(token)
  }
}

=======================================================================
[Restart auth-service, then test:]

curl -X POST http://localhost:3001/auth/verify \
-H "Authorization: Bearer YOUR_JWT_TOKEN"

========================================================

Step 3 — Wallet service: call auth-service to verify tokens
Replace the manual jwtService.verify() helper in the wallet controller with a real HTTP call to auth-service.
[wallet-service/src/wallet/wallet.controller.ts] — full replacement:

import { Controller, Get, Post, Body, Headers, BadRequestException, UnauthorizedException } from '@nestjs/common'
import { WalletService } from './wallet.service'
import axios from 'axios'

const AUTH_SERVICE_URL = 'http://localhost:3001'

@Controller('wallet')
export class WalletController {
  constructor(private walletService: WalletService) {}

  // ✅ Now delegates token verification to auth-service
  private async getUserId(authHeader: string): Promise<number> {
    if (!authHeader) throw new BadRequestException('No Authorization header provided')
    try {
      const response = await axios.post(
        `${AUTH_SERVICE_URL}/auth/verify`,
        {},
        { headers: { authorization: authHeader } },
      )
      return response.data.id
    } catch {
      throw new UnauthorizedException('Invalid or expired token')
    }
  }

  @Get('balance')
  async getBalance(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getBalance(userId)
  }

  @Post('deposit')
  async deposit(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = await this.getUserId(auth)
    if (!body.amount) throw new BadRequestException('Amount is required')
    return this.walletService.deposit(userId, body.amount)
  }

  @Post('withdraw')
  async withdraw(
    @Headers('authorization') auth: string,
    @Body() body: { amount: number },
  ) {
    const userId = await this.getUserId(auth)
    if (!body.amount) throw new BadRequestException('Amount is required')
    return this.walletService.withdraw(userId, body.amount)
  }

  @Get('transactions')
  async getTransactions(@Headers('authorization') auth: string) {
    const userId = await this.getUserId(auth)
    return this.walletService.getTransactions(userId)
  }
}

=========================================================================================

[wallet-service/src/wallet/wallet.service.ts] — remove JwtService:

import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class WalletService {
  constructor(
    private prisma: PrismaService,
    // ✅ JwtService removed — wallet no longer handles tokens
  ) {}

  async getBalance(userId: number) {
    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: {},
      create: { userId, balance: 0 },
    });
    return { balance: wallet.balance };
  }

  async deposit(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Deposit amount must be greater than 0');

    const wallet = await this.prisma.wallet.upsert({
      where: { userId },
      update: { balance: { increment: amount } },
      create: { userId, balance: amount },
    });

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'deposit', amount },
    });

    return { balance: wallet.balance };
  }

  async withdraw(userId: number, amount: number) {
    if (amount <= 0) throw new BadRequestException('Withdraw amount must be greater than 0');

    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) throw new BadRequestException('Wallet not found');
    if (wallet.balance < amount) throw new BadRequestException('Insufficient balance');

    const updated = await this.prisma.wallet.update({
      where: { userId },
      data: { balance: { decrement: amount } },
    });

    await this.prisma.transaction.create({
      data: { walletId: wallet.id, type: 'withdraw', amount },
    });

    return { balance: updated.balance };
  }

  async getTransactions(userId: number) {
    const wallet = await this.prisma.wallet.findUnique({ where: { userId } });
    if (!wallet) return [];

    return this.prisma.transaction.findMany({
      where: { walletId: wallet.id },
      orderBy: { createdAt: 'desc' },
    });
  }
}

=========================================================================================

Also remove JwtModule from wallet.module.ts since the wallet no longer needs it directly:
[wallet-service/src/wallet/wallet.module.ts]

import { Module } from '@nestjs/common'
import { WalletService } from './wallet.service'
import { WalletController } from './wallet.controller'
import { PrismaService } from '../prisma/prisma.service'

@Module({
  imports: [],                          // ✅ JwtModule removed
  providers: [WalletService, PrismaService],
  controllers: [WalletController],
})
export class WalletModule {}

======================================================================

Step 4 — Game service: call wallet-service + history-service
This is the most important connection. When a player plays a round, game-service needs to:

Deduct the bet from their wallet
Run the game logic
Pay out winnings if they won
Record the result in history-service

[game-service/src/game/game.service.ts] — full replacement:

import { Injectable, BadRequestException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import axios from 'axios'

const WALLET_SERVICE_URL = 'http://localhost:3002'
const HISTORY_SERVICE_URL = 'http://localhost:3004'

@Injectable()
export class GameService {
  constructor(private prisma: PrismaService) {}

  async createPlayer(username: string) {
    return this.prisma.player.create({
      data: { username },
    })
  }

  async getPlayers() {
    return this.prisma.player.findMany()
  }

  // ✅ Now wired to wallet + history
  async playRound(playerId: number, bet: number, authHeader: string) {
    if (bet <= 0) throw new BadRequestException('Bet must be > 0')

    // 1️⃣ Deduct bet from wallet (fails if insufficient balance)
    try {
      await axios.post(
        `${WALLET_SERVICE_URL}/wallet/withdraw`,
        { amount: bet },
        { headers: { authorization: authHeader } },
      )
    } catch (err) {
      throw new BadRequestException(
        err.response?.data?.message || 'Wallet withdraw failed'
      )
    }

    // 2️⃣ Run game logic
    const win = Math.random() < 0.5
    const payout = win ? bet * 2 : 0
    const result = win ? 'win' : 'lose'

    // 3️⃣ Pay out winnings if player won
    if (win) {
      await axios.post(
        `${WALLET_SERVICE_URL}/wallet/deposit`,
        { amount: payout },
        { headers: { authorization: authHeader } },
      )
    }

    // 4️⃣ Save round in game DB
    const round = await this.prisma.gameRound.create({
      data: { playerId, bet, result, payout },
    })

    // 5️⃣ Push stats to history-service (fire-and-forget, don't block)
    const player = await this.prisma.player.findUnique({ where: { id: playerId } })
    if (player) {
      axios.post(`${HISTORY_SERVICE_URL}/history/upsert`, {
        username: player.username,
        bets: bet,
        wins: win ? payout : 0,
        losses: win ? 0 : bet,
      }).catch(() => {})  // silent fail — history is non-critical
    }

    return round
  }

  async getRounds(playerId: number) {
    return this.prisma.gameRound.findMany({
      where: { playerId },
      orderBy: { createdAt: 'desc' },
    })
  }
}

=============================================================================================
[game-service/src/game/game.controller.ts] — pass authHeader through to the service:

import { Controller, Post, Body, Get, Param, Headers, BadRequestException } from '@nestjs/common'
import { GameService } from './game.service'

@Controller('game')
export class GameController {
  constructor(private gameService: GameService) {}

  @Post('player')
  createPlayer(@Body() body: { username: string }) {
    return this.gameService.createPlayer(body.username)
  }

  @Get('players')
  getPlayers() {
    return this.gameService.getPlayers()
  }

  // ✅ Now requires Authorization header
  @Post('play')
  playRound(
    @Headers('authorization') auth: string,
    @Body() body: { playerId: number; bet: number },
  ) {
    if (!auth) throw new BadRequestException('Authorization header required')
    return this.gameService.playRound(body.playerId, body.bet, auth)
  }

  @Get('rounds/:playerId')
  getRounds(@Param('playerId') playerId: string) {
    return this.gameService.getRounds(Number(playerId))
  }
}

==========================================================================

Step 5 — History service: no changes needed
History-service is already correct as written. It passively receives POST /history/upsert calls from game-service and accumulates stats. Nothing to change.
=============================================================================
Step 6 — Start all 4 services and test the full flow

Open 4 terminals:

[cd backend/auth-service]
npm run start:dev

[cd wallet-service]
npm run start:dev

[cd game-service]
npm run start:dev

[cd history-service] 
npm run start:dev
=====================================
Then run the full flow:

[Register]

curl -X POST http://localhost:3001/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Login]
curl -X POST http://localhost:3001/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"player@viral.com","password":"123456"}'

[Set token as variable (replace with your actual token)]
TOKEN="eyJhbGci..."

[Deposit funds]
curl -X POST http://localhost:3002/wallet/deposit \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"amount":500}'

[Create a game player]
curl -X POST http://localhost:3003/game/player \
-H "Content-Type: application/json" \
-d '{"username":"player1"}'

[Play a round (deducts bet, pays out if win, updates history)]
curl -X POST http://localhost:3003/game/play \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"playerId":1,"bet":100}'

[Check wallet balance — should reflect the result]
curl -X GET http://localhost:3002/wallet/balance \
-H "Authorization: Bearer $TOKEN"

[Check leaderboard]
curl -X GET http://localhost:3004/history/leaderboard
===================================================================================================================


