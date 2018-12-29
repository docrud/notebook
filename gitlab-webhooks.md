# 利用 gitlab webhooks 实现代码同步到 Web 服务器

## 原理
基本原理就是只要你在 gitlab 上操作了某个动作，就会触发 `webhooks`

`webhooks` 就会请求你设定好的 URL，可以通过这个 URL 作为接口触发 web 服务器操作 `git pull`


## 踩过的坑
因为 `webhooks` 请求你的接口，是模拟 `http` 请求，所以这里涉及到服务器要设置好权限

而服务器上的 apache 用户跟你登录的又不一样，所以以下设置会涉及 `apache` 用户，目前以 `www` 为例


### SSH KEY
`git pull` 是 `apache` 用户操作的，我们直接在服务器上用 ssh-keygen 命令是给你登录的当前用户创建的

所以在生成 `SSH KEY` 的命令就是

```
sudo -u www ssh-keygen -t rsa
```

然后在服务器上 `clone` 也是需要用 `apache` 用户操作，这里以 `github` 的为例。**要使用 ssh 的地址**

```
sudo -u www git clone git@github.com:docrud/notebook.git
```


### git 操作权限

基本上 `apache` 用户是不会有 git 使用权限，所以我们设定一下，命令如下：

```
vim /etc/sudoers

// 在用户 root 下方加上以下，git 的地址可以通过 which git 命令查看

www localhost=(ALL) NOPASSWD:/usr/bin/git
```

该文档是只读的，所以得强制保存


### PHP 是否开放 exec shell_exec 等函数

可以先用 `phpinfo()` 查看以下 `disable_functions`

如果被禁用了，就得通过 `php.ini` 找到 `disable_functions`，删除相对应函数名称


### webhooks

登录 gitlab，切换到项目下，`setting - Integrations` 就可以新增 `webhooks`

点 `edit` 可以查看 `Recent Deliveries`

点 `View details` 就可以查看到请求详情（用于接口的接收）

### 接口代码 demo

因为框架用的是 `thinkPHP 5.1`，所以直接在框架里写了个控制器

需求相对简单，所以代码也写得简单

```php
<?php
namespace app\git\controller;

use app\common\controller\BaseController;
use email\Email;

/**
 * Gitlab Webhooks
 * 
 * chao@DoCRUD
 */
class Index extends BaseController
{
    // 在 gitlab webhooks 设定好的 Secret Token
    private const ACCESS_TOKEN = 'xxxxxxxxxxxxxxxxxxxx';

    // 在 gitlab webhooks 里设定好的操作
    private const ACCESS_EVENT = 'Merge Request Hook';

    public function index()
    {
        // 获取到 gitlab webhooks 请求头部
        $token = $this->request->header('X_GITLAB_TOKEN');
        $event = $this->request->header('X_GITLAB_EVENT');

        // 邮件通知
        $mail = new Email();
        $mail->sendTo('chao@docrud.com');
        $body = 'Request on [' . date('Y-m-d H:i:s') . '] from [' . $this->request->ip() . ']';

        // Token 验证
        if ($token !== self::ACCESS_TOKEN) {
            $subject = 'Invalid token [' . $token . ']';
            $mail->sendText($subject, $body);

            return $this->buildFailed('401', 'Invalid token');
        }

        // 触发事件验证
        if ($event !== self::ACCESS_EVENT) {
            $subject = 'Invalid event [' . $event . ']';
            $mail->sendText($subject, $body);

            return $this->buildFailed('403', 'Invalid event');
        }

        // 操作限定，只在 merge 操作才触发 git pull
        $attributes = $this->request->param('object_attributes');
        $mergeStatus = $attributes['merge_status'];

        // 判断操作状态
        if ($mergeStatus != 'can_be_merged') {
            $subject = 'Invalid merge status [' . $mergeStatus . ']';
            $mail->sendText($subject, $body);

            return $this->buildFailed('403', 'Invalid merge status');
        }

        // 先切换到项目目录下
        exec("cd /home/wwwroot/docrud");

        // 执行 git pull
        exec("git pull origin master 2>&1", $output, $status);

        $subject = 'Merge status: ' . $mergeStatus;
        $body .= '<br>Pull result: ' . json_encode($output);
        $mail->sendText($subject, $body);

        return $this->buildSuccess();
    }
}
```
