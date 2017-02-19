---
id: do-not-confuse-env-for-deploy-target-ru
title: Не путайте environment с deploy target
summary: У большинства фрэймворков есть понятие *environment*. Многие разработчики привязывают к environment параметры разветывания, что неверно. 
date: 2017-02-18
---

Недавно я в очередной раз наткнулся на эту проблему и решил написать о ней данную заметку. Пишу с оглядкой на фронтенд-фрэймворк **EmberJS**, но суть применима к любому фронтенд- и бэкенд-фрйэмворку.



## Термин environment слишком размыт

Термин **environment** применяется очень широко, что вызывает путаницу. В интернете есть много статей, которые, если прочитать их поверхностно, противоречат тому, что я хочу посоветовать ниже. Долгое время эти статьи вызывали у меня недоумение. Пораскинув мозгами, я понял, что дело в путаной терминологии.

В **EmberJS** и многих других вэб-фрэймворках понятие **environment** применяется в достаточно узком смысле, и я далее буду придеживаться именно его. **Environment** — это набор параметров, определяющих параметры сборки и запуска вэб-приложения:

*   минификация кода,
*   asset fingerprinting/cache busting,
*   генерация source maps,
*   внедрение различных средств, облегчающих отладку,
*   тестирование, внедрение маркеров для оценки покрытия кода.

Но есть и другая группа параметров, которую также принято включать в понятие **environment**. Я считаю, что это большая ошибка, и предлагаю отличать две группы. Вторую группу параметров я называю **deploy target**:

*   адреса API,
*   адреса CDN,
*   настройки CSP (content security policy) и CORS (cross-origin resource sharing),
*   ключи API.



## В чем, собственно, проблема

Многие разработчики сваливают обе группы параметров в одну кучу и привязывают к параметру **environment**, который у большинства фрэймворков может принимать всего три значения: `development`, `production` и `testing`. В результате приложение по сути имеет всего два режима сборки: production-билд, привязанный к production серверу, и development-билд привязанный к локальному серверу или моку (имитации сервера).

Сделать сборку в development-режиме, но привязанную к production серверу, затруднительно: для этого приходится временно редактировать конфиг. А такая сборка может очень выручить, когда нужно отладить пробему, которая проявляется в продакшене, но не воспроизводится локально.

Обратное тоже нередко необходимо: сделать сборку для локального сервера, но в production режиме. Это нужно, чтобы замерить производительность или чтобы отладить проблему, возникающую во время минификации и фингерпринтинга.

А хуже всего то, что ключи API хранятся прямо в конфиге и попадают в систему контроля версий, что угрожает безопасности вашего сервиса.



## Как надо делать

Моя рекомендация сводится к следующему:

1.  Удалите параметры группы **deploy target** из конфига приложения.

2.  Вынесите их в отдельные файлы, по одному для каждого сервера: production, staging, development, sandbox, local, mock... Их может быть столько, сколько вам нужно, а не только production и development.

3.  Файлы с параметрами **deploy target** обязательно добавьте в `.gitignore`, чтобы ключи API не засветились в системе контроля версий.

4.  Если вы используете CI (continuous integration), содержимое этих файлов можно скопировать в настройки каждого сервера. На CodeShip, например, это называется "deployment pipelines".

5.  Настройте приложение так, чтобы при сборке можно было указать, какой **deploy target** использовать.

Теперь **deploy target** можно выбирать отдельно от **environment**, получая любое нужное вам сочетание **deploy target** и **environment**. Например, я это делаю так:

    DEPLOY_TARGET=local ember serve --environment=production

или просто

    DEPLOY_TARGET=local ember s -prod

Разумеется, для удобства вы можете настроить, чтобы для каждого **environment** автоматически использовался определенный **deploy target**, чтобы не приходилось указывать каждый раз. Главное, чтобы в нужный момент можно было переопределить без необходимости редактировать код, а потом откатывать.



## Как этого добиться в Ember

Параметры **deploy target** мы будем передавать через переменные окружения, т. е. **environment variables**. Название этих переменных добавляет путаницы, так что будьте внимательны.

Хранить параметры **deploy target** в файлах нам поможет аддон [ember-cli-dotenv](https://github.com/fivetanley/ember-cli-dotenv).

Создайте в корневой папке вашего Ember-приложения `.env`-файлы, по одному для каждого вашего сервера (включая локальный и мок-сервер):

    .env-prod
    .env-staging
    .env-sandbox
    .env-local
    .env-mock

Особенно круто, если у ваших серверов есть имена, отличающиеся от production/staging/development. В таком случае используйте эти имена в названиях `.env`-файлов.

В каждом файле храните параметры **deploy target** в таком виде:

    ABC_BACKEND_API_URL=https://bravo.horns-and-hoofs.com/api
    ABC_BACKEND_API_VERSION=v18
    ABC_IMAGES_CDN_URL=http://horns-and-hoofs.cloudfront.net/bravo/images
    ABC_GITHUB_API_KEY=jFViG9kZtY4NAJA8I65s

`ABC` в данном случае — это аббревиатура названия вашего приложения. Добавлять его полезно, чтобы не переопределить какую-либо внешнюю переменную, которая может потребоваться и навзание которой может совпадать с одним из ваших.

Теперь нам нужно добиться, чтобы `ember-cli-dotenv` подгружал нужный `.env`-файл. Для этого воспользуйтесь таким трюком в файле `ember-cli-build.js`:

```javascript
// Написано с расчетом на современную версию NodeJS
const dotEnvFileName = (() => {
  const environment   = process.env.EMBER_ENV || 'development';
  const targetDefault = environment === 'production' ? 'prod' : 'local'; // Укажите значения по умолчанию
  const target        = process.env.ABC_DEPLOY_TARGET || defaultTarget;
  const dotEnvFile    = `./.env-${deployTarget}`;

  if (!fs.existsSync(dotEnvFile)) {
    throw new Error(`dot-env file not found: ${dotEnvFile} for DEPLOY_TARGET ${target}`);
  }

  return dotEnvFile;
})();



module.exports = function (defaults) {
  const app =
    new EmberApp(defaults, {

      // Передаем нужный файл в ember-cli-dotenv
      dotEnv: {
        clientAllowedKeys: [/* перечислите имена ключей из .env-файла */],
        path: dotEnvFile
      },
     
// ...
```

Теперь параметры из `.env`-файла попадут в хэш `process.env`, который вы можете использовать в `config/environment.js` следующим образом:

```javascript
{
  gitHubApiKey: process.env.ABC_GITHUB_API_KEY
}
```

Еще я люблю создать в Ember-приложении сервис `config` и считывать все параметры из этого сервиса, а не напрямую из `config/environment.js`. Это дает возможность использовать computed properties для формирования производных параметров, например, добавления версии API в адрес сервера.

Использовать можно так:

    ABC_DEPLOY_TARGET=local ember s -prod

Если же вам нужна сборка для production-сервера в режиме production или сборка для локального сервера в режиме development, то можно полагаться на значения по умолчанию:
 
     ember s -prod
     ember s



## Ваше мнение?

Обязательно поделитесь своими соображениями в комментариях!