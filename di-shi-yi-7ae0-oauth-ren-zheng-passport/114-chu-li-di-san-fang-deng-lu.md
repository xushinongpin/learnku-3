## 处理第三方登录

回忆一下第四章[第三方登录](https://learnku.com/courses/laravel-advance-training/5.5/796/process-explanation)的内容，第三方登录的最后，在服务器端我们通过微信的`access_token`换取了微信的用户信息，最后生成我们自己的令牌信（JWT），发送给客户端。现在我们需要将最后生成令牌的方式由的 JWT 替换为 Passport。

## 个人访问令牌

Passport 为我们提供了一种直接某个用户生成令牌的方式 ——[个人访问令牌](https://learnku.com/docs/laravel/5.5/passport#personal-access-tokens)。

首先需要创建一个`personal`的客户端`php artisan passport:client --personal`，然后我们就可以直接通过命令`$token = $user->createToken('Token Name')->accessToken;`创建一个访问令牌。

这种创建令牌的方式有两个问题

* 只能为用户创建
  `access_token`
  ，没有
  `refresh_token`
  ；
* 个人访问令牌是永久有效的，就算使用了
  `tokensExpireIn`
  和
  `refreshTokensExpireIn`
  方法也不会修改它的生命周期；

第三方登录应该同登录接口一样，有完整的令牌信息返回，包括`refresh_token`，这样才能在客户端保持用户一直登录；永久有效的令牌显然是也是不安全的，所以通过通过这种方式创建的令牌显然不是我们想要的。接下来我们将寻找更加合理的解决方案。

## 直接生成访问令牌

绕过 Oauth2 的授权流程，我们能否直接为某个用户生成完整的令牌信息呢？我们在 Passport 的 GitHub Issues 中找到了[解决方法](https://github.com/laravel/passport/issues/71#issuecomment-330506407)。

先来创建一个 Trait：

```
$ touch app/Traits/PassportToken.php
```

_app/Traits/PassportToken.php_

```
<?php

namespace App\Traits;

use App\Models\User;
use DateTime;
use GuzzleHttp\Psr7\Response;
use Illuminate\Events\Dispatcher;
use Laravel\Passport\Bridge\AccessToken;
use Laravel\Passport\Bridge\AccessTokenRepository;
use Laravel\Passport\Bridge\Client;
use Laravel\Passport\Bridge\RefreshTokenRepository;
use Laravel\Passport\Passport;
use Laravel\Passport\TokenRepository;
use League\OAuth2\Server\CryptKey;
use League\OAuth2\Server\Entities\AccessTokenEntityInterface;
use League\OAuth2\Server\Exception\OAuthServerException;
use League\OAuth2\Server\Exception\UniqueTokenIdentifierConstraintViolationException;
use League\OAuth2\Server\ResponseTypes\BearerTokenResponse;

# https://github.com/laravel/passport/issues/71

trait PassportToken
{
    private function generateUniqueIdentifier($length = 40)
    {
        try {
            return bin2hex(random_bytes($length));
            // @codeCoverageIgnoreStart
        } catch (\TypeError $e) {
            throw OAuthServerException::serverError('An unexpected error has occurred');
        } catch (\Error $e) {
            throw OAuthServerException::serverError('An unexpected error has occurred');
        } catch (\Exception $e) {
            // If you get this message, the CSPRNG failed hard.
            throw OAuthServerException::serverError('Could not generate a random string');
        }
        // @codeCoverageIgnoreEnd
    }

    private function issueRefreshToken(AccessTokenEntityInterface $accessToken)
    {
        $maxGenerationAttempts = 10;
        $refreshTokenRepository = app(RefreshTokenRepository::class);

        $refreshToken = $refreshTokenRepository->getNewRefreshToken();
        $refreshToken->setExpiryDateTime((new \DateTime())->add(Passport::refreshTokensExpireIn()));
        $refreshToken->setAccessToken($accessToken);

        while ($maxGenerationAttempts-- > 0) {
            $refreshToken->setIdentifier($this->generateUniqueIdentifier());
            try {
                $refreshTokenRepository->persistNewRefreshToken($refreshToken);

                return $refreshToken;
            } catch (UniqueTokenIdentifierConstraintViolationException $e) {
                if ($maxGenerationAttempts === 0) {
                    throw $e;
                }
            }
        }
    }

    protected function createPassportTokenByUser(User $user, $clientId)
    {
        $accessToken = new AccessToken($user->id);
        $accessToken->setIdentifier($this->generateUniqueIdentifier());
        $accessToken->setClient(new Client($clientId, null, null));
        $accessToken->setExpiryDateTime((new DateTime())->add(Passport::tokensExpireIn()));

        $accessTokenRepository = new AccessTokenRepository(new TokenRepository(), new Dispatcher());
        $accessTokenRepository->persistNewAccessToken($accessToken);
        $refreshToken = $this->issueRefreshToken($accessToken);

        return [
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken,
        ];
    }

    protected function sendBearerTokenResponse($accessToken, $refreshToken)
    {
        $response = new BearerTokenResponse();
        $response->setAccessToken($accessToken);
        $response->setRefreshToken($refreshToken);

        $privateKey = new CryptKey('file://'.Passport::keyPath('oauth-private.key'), null, false);

        $response->setPrivateKey($privateKey);
        $response->setEncryptionKey(app('encrypter')->getKey());

        return $response->generateHttpResponse(new Response);
    }

    protected function getBearerTokenByUser(User $user, $clientId, $output = true)
    {
        $passportToken = $this->createPassportTokenByUser($user, $clientId);
        $bearerToken = $this->sendBearerTokenResponse($passportToken['access_token'], $passportToken['refresh_token']);

        if (! $output) {
            $bearerToken = json_decode($bearerToken->getBody()->__toString(), true);
        }

        return $bearerToken;
    }
}
```

代码有点多，不过我们只需要注意`getBearerTokenByUser`方法即可，该方法提供三个参数

* 用户实例；
* Passport 客户端的 client\_id；
* 是否直接返回 Response；

_app/Http/Controllers/Api/AuthorizationsController.php_

```
.
.
.
use App\Traits\PassportToken;

class AuthorizationsController extends Controller
{
    use PassportToken;
.
.
.
```

在`AuthorizationsController`中增加`PassportToken`Trait，这样我们在控制器中就可以直接通过`getBearerTokenByUser`方法直接为某个用户生成完整的访问令牌。

## 调整第三方登录接口

```
.
.
.
public function socialStore($type, SocialAuthorizationRequest $request)
{
.
.
.
    $result = $this->getBearerTokenByUser($user, '1', false);
    return $this->response->array($result)->setStatusCode(201);
}
.
.
.
```

只需要调整最后生成令牌的代码即可。

## 使用 PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/rn33hBd411.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/rn33hBd411.png)

通过微信开发者工具制作一个用户授权码，如果你忘记了如何获取授权码，可以看[这里](https://learnku.com/courses/laravel-advance-training/5.5/798/wechat-registration#c078aa)回忆一下。可以正确的获取完整的令牌信息。

## 代码版本控制

```
$ git add -A
$ git commit -m 'social login change to Passport'
```



