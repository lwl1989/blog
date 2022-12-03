CreateTime:2020-08-24 15:48:22.0

```
[program:translation-en-correct-worker]
command=/bin/bash -c 'do sth'
autostart=true
autorestart=true
user=ubuntu
numprocs=1
redirect_stderr=true
stdout_logfile=/home/ubuntu/sentence_correct_v2/worker.log
```

sudo supervisorctl reload

sudo supervisorctl start translation-en-correct-worker