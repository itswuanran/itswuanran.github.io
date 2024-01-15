---
title: 理解csrf
date: 2018-01-15 11:07:18
tags: [CSRF, HTTP]
categories:
- HTTP
---

## csrf例子
假如一家銀行用以執行轉帳操作的URL地址如下： http://www.examplebank.com/withdraw?account=AccoutName&amount=1000&for=PayeeName

那麼，一個惡意攻擊者可以在另一个網站上放置如下代碼： <img src="http://www.examplebank.com/withdraw?account=Alice&amount=1000&for=Badman”>

这里需要防止csrf攻击的是银行的网站

如果有賬戶名為Alice的用戶訪問了惡意站點，而她之前剛訪問過銀行不久，登錄信息尚未過期，那麼她就會損失1000資金。

這種惡意的網址可以有很多種形式，藏身於網頁中的許多地方。此外，攻擊者也不需要控制放置惡意網址的網站。例如他可以將這種地址藏在論壇，博客等任何用戶生成內容的網站中。這意味著如果伺服器端沒有合適的防禦措施的話，用戶即使訪問熟悉的可信網站也有受攻擊的危險。

透過例子能夠看出，攻擊者並不能通過CSRF攻擊來直接獲取用戶的帳戶控制權，也不能直接竊取用戶的任何信息。他們能做到的，是欺騙用戶瀏覽器，讓其以用戶的名義執行操作。


## 预防
在请求时每次都验证下csrf_token的正确性，以此过滤跨域请求（跨域请求是拿不到准确的csrf_token的）。
那么如何获取csrf_token呢？每次请求通过$request->getCsrfToken获取token，第一次请求时token生成后放在cookie中（不可用放session）然后前端生成的maskcsrftoken，然后请求时后端验证maskcsrftoken和masktruetoken。

参见yii/web/Request.php
```php
    /**
     * @inheritdoc
     */
    public function beforeAction($action)
    {
        if (parent::beforeAction($action)) {
            if ($this->enableCsrfValidation && Yii::$app->getErrorHandler()->exception === null && !Yii::$app->getRequest()->validateCsrfToken()) {
                throw new BadRequestHttpException(Yii::t('yii', 'Unable to verify your data submission.'));
            }

            return true;
        }

        return false;
    }

    /**
     * Returns the token used to perform CSRF validation.
     *
     * This token is generated in a way to prevent [BREACH attacks](http://breachattack.com/). It may be passed
     * along via a hidden field of an HTML form or an HTTP header value to support CSRF validation.
     * @param bool $regenerate whether to regenerate CSRF token. When this parameter is true, each time
     * this method is called, a new CSRF token will be generated and persisted (in session or cookie).
     * @return string the token used to perform CSRF validation.
     */
    public function getCsrfToken($regenerate = false)
    {
        if ($this->_csrfToken === null || $regenerate) {
            if ($regenerate || ($token = $this->loadCsrfToken()) === null) {
                $token = $this->generateCsrfToken();
            }
            $this->_csrfToken = Yii::$app->security->maskToken($token);
        }

        return $this->_csrfToken;
    }

    /**
     * Loads the CSRF token from cookie or session.
     * @return string the CSRF token loaded from cookie or session. Null is returned if the cookie or session
     * does not have CSRF token.
     */
    protected function loadCsrfToken()
    {
        if ($this->enableCsrfCookie) {
            return $this->getCookies()->getValue($this->csrfParam);
        }

        return Yii::$app->getSession()->get($this->csrfParam);
    }

    /**
     * Generates an unmasked random token used to perform CSRF validation.
     * @return string the random token for CSRF validation.
     */
    protected function generateCsrfToken()
    {
        $token = Yii::$app->getSecurity()->generateRandomString();
        if ($this->enableCsrfCookie) {
            $cookie = $this->createCsrfCookie($token);
            Yii::$app->getResponse()->getCookies()->add($cookie);
        } else {
            Yii::$app->getSession()->set($this->csrfParam, $token);
        }

        return $token;
    }

    /**
     * @return string the CSRF token sent via [[CSRF_HEADER]] by browser. Null is returned if no such header is sent.
     */
    public function getCsrfTokenFromHeader()
    {
        return $this->headers->get(static::CSRF_HEADER);
    }

    /**
     * Creates a cookie with a randomly generated CSRF token.
     * Initial values specified in [[csrfCookie]] will be applied to the generated cookie.
     * @param string $token the CSRF token
     * @return Cookie the generated cookie
     * @see enableCsrfValidation
     */
    protected function createCsrfCookie($token)
    {
        $options = $this->csrfCookie;
        return Yii::createObject(array_merge($options, [
            'class' => 'yii\web\Cookie',
            'name' => $this->csrfParam,
            'value' => $token,
        ]));
    }

    /**
     * Performs the CSRF validation.
     *
     * This method will validate the user-provided CSRF token by comparing it with the one stored in cookie or session.
     * This method is mainly called in [[Controller::beforeAction()]].
     *
     * Note that the method will NOT perform CSRF validation if [[enableCsrfValidation]] is false or the HTTP method
     * is among GET, HEAD or OPTIONS.
     *
     * @param string $clientSuppliedToken the user-provided CSRF token to be validated. If null, the token will be retrieved from
     * the [[csrfParam]] POST field or HTTP header.
     * This parameter is available since version 2.0.4.
     * @return bool whether CSRF token is valid. If [[enableCsrfValidation]] is false, this method will return true.
     */
    public function validateCsrfToken($clientSuppliedToken = null)
    {
        $method = $this->getMethod();
        // only validate CSRF token on non-"safe" methods https://tools.ietf.org/html/rfc2616#section-9.1.1
        if (!$this->enableCsrfValidation || in_array($method, ['GET', 'HEAD', 'OPTIONS'], true)) {
            return true;
        }

        $trueToken = $this->getCsrfToken();

        if ($clientSuppliedToken !== null) {
            return $this->validateCsrfTokenInternal($clientSuppliedToken, $trueToken);
        }

        return $this->validateCsrfTokenInternal($this->getBodyParam($this->csrfParam), $trueToken)
            || $this->validateCsrfTokenInternal($this->getCsrfTokenFromHeader(), $trueToken);
    }

    /**
     * Validates CSRF token.
     *
     * @param string $clientSuppliedToken The masked client-supplied token.
     * @param string $trueToken The masked true token.
     * @return bool
     */
    private function validateCsrfTokenInternal($clientSuppliedToken, $trueToken)
    {
        if (!is_string($clientSuppliedToken)) {
            return false;
        }

        $security = Yii::$app->security;

        return $security->unmaskToken($clientSuppliedToken) === $security->unmaskToken($trueToken);
    }


```

## 理解
黑客：A.com 银行：B.com  用户：C
A通过curl或者其他方式访问B获取到的csrf token只是针对用户A的，对C来说是无影响的。

## 展望
因为web正在向JSON API转移，并且浏览器变得更安全，有更多的安全策略， CSRF正在变得不那么值得关注。 阻止旧的浏览器访问你的站点，并尽可能的将你的API变成JSON API， 然后你将不再需要CSRF token。 但是为了安全起见，你还是应该尽量允许他们尤其是当难以实现的时候。

## 参考
- https://github.com/pillarjs/understanding-csrf/blob/master/README_zh.md