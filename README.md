# Trade-DB
Схема базы данных для реализации минимальной функциональности финансовой биржи

## Примеры бизнес-задач которые решает база
- Регистрация, авторизация пользователя(users) (RBAC опущена, в интернете множество реализаций)
- Создание платежных аккаунтов(accounts)(для различных валют(curriencies) или тарифов(tariffs))
- Выставление счетов(payments) для зачисления средств в аккаунт(accounts)
- Тарифы(tariffs) для аккаунтов(accounts), в тарифах указаны доступные финансовые инструменты(instruments) и комиссия сервиса за сделку(orders)
- Заказы(orders) это заявка на покупку или продажу(type) определенного финансового инструмента по определенной цене(amount)
- Инструменты(instruments) - это финансовые инструменты, будь то бонды, акции, облигации(type), сделки по ним могут осуществлятся только в определенной валюте(curriencies) и указаны минимальный объем для покупки(lot)

## Схема базы данных
- ![Схема](https://github.com/orensimple/trade-db/edit/main/schema.png?raw=true)
- Поля у которых тип text это поля для которых не созданы справочники, что бы не усложнять схему

## Документация
- В качестве первичного ключа(id) используется uuid(с точки зрения безопасности и анонимности данных)

- Индексы сделаны на колонки code для валют, инструментов, и тарифов (поиск по коду, константе)
- Полнотекстовый индекс в таблице пользователей(users) на поиск пользователя по префиксу имени и фамилии (необходим в рамках задачи по курсу Highloud)
- Индекс в таблице пользователей(users) по email для авторизации и регистрации
- Индекс по таблице инструментов(instruments) для поиска по определенному инструменту заказов со статусом новый

``` sql
create table currencies
(
    id           varchar(36)     not null,
    code         char(3)         not null,
    name         varchar(36)     not null,
    multiplicity int default 100 not null comment 'for kopecks and cents',
    constraint currencies_code_uindex
        unique (code),
    constraint currencies_id_uindex
        unique (id)
)
    comment 'Table of currencies used';

create index currencies_code_index
    on currencies (code);

alter table currencies
    add primary key (id);

create table instruments
(
    id          varchar(36)                         not null,
    currency_id varchar(36)                         not null,
    code        varchar(16)                         not null,
    name        varchar(32)                         not null,
    description varchar(256)                        null,
    lot         int                                 not null comment 'the minimum quantity that can be sold at a time',
    type        text                                not null comment 'the type of financial instrument, for example - a currency pair, stocks, abligations, bonds (for simplicity, I did not put it into the directory)',
    created_at  timestamp default CURRENT_TIMESTAMP not null,
    updated_at  timestamp                           null,
    deleted_at  timestamp                           null,
    constraint instrument_code_uindex
        unique (code),
    constraint instrument_id_uindex
        unique (id),
    constraint instruments_currencies_id_fk
        foreign key (currency_id) references currencies (id)
);

create index instruments_code_index
    on instruments (code);

alter table instruments
    add primary key (id);

create table tariffs
(
    id         varchar(36)                             not null,
    code       varchar(16)                             not null,
    name       varchar(256)                            not null,
    commission decimal(3, 2) default 0.00              not null comment 'service commission percentage',
    created_at timestamp     default CURRENT_TIMESTAMP not null,
    updated_at timestamp                               null,
    deleted_at timestamp                               null,
    constraint tariffs_code_uindex
        unique (code),
    constraint tariffs_id_uindex
        unique (id)
)
    comment 'Custom rates table';

create index tariffs_code_index
    on tariffs (code);

alter table tariffs
    add primary key (id);

create table accounts
(
    id          varchar(36)                             not null,
    currency_id varchar(36)                             not null,
    tariff_id   varchar(36)                             not null,
    name        varchar(256)                            not null,
    balance     decimal(3, 2) default 0.00              not null comment 'сurrent balance',
    created_at  timestamp     default CURRENT_TIMESTAMP not null,
    updated_at  timestamp                               null,
    deleted_at  timestamp                               null,
    constraint accounts_id_uindex
        unique (id),
    constraint accounts_currencies_id_fk
        foreign key (currency_id) references currencies (id),
    constraint accounts_tariffs_id_fk
        foreign key (tariff_id) references tariffs (id)
)
    comment 'User billing table';

alter table accounts
    add primary key (id);

create table orders
(
    id            varchar(36)                         not null,
    account_id    varchar(36)                         not null,
    instrument_id varchar(36)                         not null,
    type          varchar(64)                         not null comment 'type of order, for example: buy, sell (for simplicity, I did not put it in the directory)',
    price         decimal                             not null comment 'the price at which the user wants to purchase the instrument',
    volume        int                                 not null comment 'number of lots',
    status        varchar(64)                         null comment 'order status, for example: completed, canceled (for simplicity, I did not put it into the directory)',
    created_at    timestamp default CURRENT_TIMESTAMP not null,
    updated_at    timestamp                           null,
    constraint orders_order_id_uindex
        unique (id),
    constraint orders_accounts_id_fk
        foreign key (account_id) references accounts (id),
    constraint orders_instruments_id_fk
        foreign key (instrument_id) references instruments (id)
)
    comment 'User order table';

create index orders_account_id_instrument_id_type_index
    on orders (account_id, instrument_id, type);

create index orders_instrument_id_type_status_created_at_index
    on orders (instrument_id, type, status, created_at);

alter table orders
    add primary key (id);

create table payments
(
    id           varchar(36)                         not null,
    account_id   varchar(36)                         not null,
    amount       decimal                             not null comment 'amount to be credited to the account',
    bank_details text                                null comment 'bank details, can be divided into account number, INN, KPP of the bank',
    created_at   timestamp default CURRENT_TIMESTAMP not null,
    paid_at      timestamp                           null,
    constraint payments_id_uindex
        unique (id),
    constraint payments_accounts_id_fk
        foreign key (account_id) references accounts (id)
)
    comment 'Invoice table';

alter table payments
    add primary key (id);

create table tariff_instruments
(
    id            varchar(36) not null,
    tariff_id     varchar(36) not null,
    instrument_id varchar(36) not null,
    primary key (tariff_id, instrument_id),
    constraint tariff_instruments_id_uindex
        unique (id),
    constraint tariff_instruments_instruments_id_fk
        foreign key (instrument_id) references instruments (id),
    constraint tariff_instruments_tariffs_id_fk
        foreign key (tariff_id) references tariffs (id)
);

create table users
(
    id         varchar(36)                         not null,
    email      varchar(64)                         not null,
    password   varchar(64)                         not null,
    first_name varchar(32)                         not null,
    last_name  varchar(32)                         not null,
    passport   int                                 null comment 'series and number in the format 5612453765',
    male       tinyint(1)                          null comment 'man is true, a woman is false, not selected null',
    address    varchar(256)                        null,
    about      varchar(256)                        null comment 'additional information',
    created_at timestamp default CURRENT_TIMESTAMP not null,
    updated_at timestamp                           null,
    deleted_at timestamp                           null,
    constraint users_id_uindex
        unique (id)
    constraint users_email_uindex
        unique (email),
)
    comment 'User data table';

create index user_email_idx
    on users (email);

create fulltext index user_first_name_last_name_ftsidx
    on users (first_name, last_name);

alter table users
    add primary key (id);

create table user_accounts
(
    id         varchar(36) not null,
    user_id    varchar(36) not null,
    account_id varchar(36) not null,
    primary key (user_id, account_id),
    constraint user_accounts_id_uindex
        unique (id),
    constraint user_accounts_accounts_id_fk
        foreign key (account_id) references accounts (id),
    constraint user_accounts_users_id_fk
        foreign key (user_id) references users (id)
);

```

## Репликация
В процессе разработки

## Резервное копирование
В процессе разработки
