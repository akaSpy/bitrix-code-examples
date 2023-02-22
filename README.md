# Пример кода в системе на базе Bitrix Framework

## Структура компонента

```
/local/components/auth/
    ├── class.php
    └── templates
        └── b2c
            ├── dist
            ├── lang
            ├── src
            │   ├── Components
            │   │   ├── Login.vue
            │   │   ├── LoginByEmail.vue
            │   │   ├── LoginByPhone.vue
            │   │   ├── RegisterByEmail.vue
            │   │   ├── RegisterByEmailConfirm.vue
            │   │   ├── RegisterByEmailMessage.vue
            │   │   ├── RegisterByPhone.vue
            │   │   ├── RestorePassword.vue
            │   │   └── RestorePasswordMessage.vue
            │   ├── api.js
            │   └── index.js
            ├── bundle.config.js
            ├── style.css
            └── template.php

# Структура по типу package-by-feature (это не совсем из реального проекта, 
# но примерно показано к чему я бы стремился)

/local/php_interface/src/
    └── Features
        ├── Auth
        │   ├── Entities 
        │   │   ├── User.php
        │   │   ├── UserEmail.php
        │   │   └── ...
        │   ├── Repositories
        │   │   └── UserRepository.php
        │   └── Services
        │       └── Tokenizer.php
        └── Profile
```

### class.php

```php
<?php

declare(strict_types=1);

namespace Local\Components\Common;

use App\Features\Auth\Entities\User;
use App\Features\Auth\Entities\UserEmail;
use App\Features\Auth\Repositories\UserRepository;
use Bitrix\Main\Error;

class AuthComponent extends \CBitrixComponent implements Controllerable
{
    // ...

    /**
     * Вход по адресу электронной почты.
     */
    public function loginByEmailAction($email, $password, UserRepository $users): AjaxJson
    {
        global $USER;
    
        try {
            $email = new UserEmail($email);
    
            if (!$user = $users->findByEmail($email)) {
                return $this->error([new Error('Пользователь с таким email не найден.', 'user-not-found')]);
            }
    
            if ($user->isNotActive()) {
                return $this->error([new Error('Неверный логин или пароль.', 'access')]);
            }
    
            $result = $USER->Login($email->getValue(), $password, 'Y');
    
            // Важно, чтобы условие было $result !== true,
            // это единственный признак успешности авторизации.
            if ($result !== true) {
                return $this->error([new Error('Неверный логин или пароль.', 'access')]);
            }
    
            return $this->ok();
        } catch (\InvalidArgumentException $e) {
            return $this->error([new Error($e->getMessage(), 'validation')]);
        } catch (\Exception $e) {
            return $this->error([new Error('Произошла системная ошибка.', 'system')]);
        }
    }
    
    // ...
    
    /**
     * @param list<Error> $errors
     */
    private function error(array $errors): AjaxJson
    {
        return AjaxJson::createError(new ErrorCollection($errors));
    }
    
    private function ok(): AjaxJson
    {
        return AjaxJson::createSuccess();
    }
}
```

### User.php

```php
<?php

declare(strict_types=1);

namespace App\Features\Auth\Entities\User;

class User
{
    private UserId $id;
    private bool $active = false;
    private ?UserEmail $email = null;
    private ?UserEmail $newEmail = null;
    private bool $isEmailConfirmed = false;
    private ?UserPassword $password = null;
    private EmailConfirmationToken $emailConfirmationToken;
    private PhoneConfirmationCode $phoneConfirmationCode;

    public function changeEmailRequest(UserEmail $newEmail, EmailConfirmationToken $emailConfirmationToken): void
    {
        if (empty($this->email)) {
            throw new \DomainException('У пользователя не указан текущий Email.');
        }

        if ($this->isRequestToChangeEmailSent()) {
            throw new \DomainException('Запрос на смену Email уже был отправлен.');
        }

        if ($this->email->isEqualTo($newEmail)) {
            throw new \DomainException('Новый Email не может быть равен текущему.');
        }

        if ($this->isEmailNotConfirmed()) {
            throw new \DomainException('Подтвердите операцию по смене Email по ссылке из письма. Или отмените изменения.');
        }

        $this->newEmail = $newEmail;

        $this->resetEmailConfirmationToken($emailConfirmationToken);
    }

    public function changeEmailRollback($token, EmailConfirmationToken $newToken): void
    {
        if (empty($this->newEmail) || !$this->isRequestToChangeEmailSent()) {
            throw new \DomainException('Запрос на смену Email не был отправлен.');
        }

        if (empty($this->emailConfirmationToken) || !$this->emailConfirmationToken->isValidTo($token)) {
            throw new \DomainException('Неверный токен.');
        }

        $this->resetEmailConfirmationToken($newToken);
    }
    
    public function isNotActive(): bool
    {
        return $this->active === false;
    }
    
    // ...
}
```

### UserRepository.php

```php
<?php

declare(strict_types=1);

namespace App\Features\Auth\Repositories\UserRepository;

class UserRepository
{
    private $hydrator;

    public function __construct()
    {
        $this->hydrator = new Hydrator();
    }
    
    public function findByEmail(UserEmail $email): ?User
    {
        $result = \CUser::GetList(
            ($by = 'id'),
            ($order = 'desc'),
            ['=EMAIL' => $email->getValue()],
            [
                'SELECT' => [
                    'ID',
                    'ACTIVE',
                    'EMAIL',
                    'PHONE_NUMBER',
                    'UF_CHECKWORD',
                    'UF_SMS_CODE',
                    'UF_SMS_CODE_TIME',
                ],
                'NAV_PARAMS' => [
                    'nTopCount' => 1
                ],
            ]
        );

        $current = $result->fetch();

        if (empty($current) || !isset($current['ID']) || empty($current['ID'])) {
            return null;
        }

        $user = $this->buildUserModel([
            'id' => $current['ID'],
            'active' => $current['ACTIVE'] === 'Y',
            'email' => $current['EMAIL'],
            'emailConfirmationToken' => $current['UF_CHECKWORD'],
            'phoneConfirmationCode' => $current['UF_SMS_CODE'],
            'phoneConfirmationCodeExpires' => $current['UF_SMS_CODE_TIME'],
        ]);

        return $user;
    }
    
    // ...
}
```

### LoginByEmail.vue

```vue
<template>
    ...
</template>

<script>
import Api from "../Api.js";

export default {
    name: "LoginByEmail",
    
    data() {
      return {
        password: '',
        backendError: '',
      };
    },
    
    // ...
    
    authorizeUserViaEmailRedirectOrFail(email, password) {
      let self = this;
      this.Api
        .loginByEmail(email, password)
        .then(
          function (response) {
            window.location.replace(window.location.origin + window.location.pathname);
          },
          function (response) {
            response.errors.forEach(function (error) {
              self.backendError = error.message;
            });
          },
        )
      ;
    },
}
</script>
```
