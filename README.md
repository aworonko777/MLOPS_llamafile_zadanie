# MLOPS_llamafile_zadanie

# Uruchomienie modelu llamafile jako usługi systemd


## 1. Cel

Model językowy `llamafile` działa jako usługa systemd pod osobnym użytkownikiem systemowym. REST API jest dostępne z gospodarza przez adres host-only `192.168.128.2:8080` i niedostępne przez pozostałe adresy maszyny.

## 2. Pobranie llamafile

Wymagania:
- włączony ufw
- maszyna z interfejsem enp0s2
- działający SSH

Plik jest pobrany przez polecenie curl na maszynę, ponieważ ma dostęp do internetu z repozytorium https://github.com/mozilla-ai/llamafile.

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin llamafile
sudo mkdir -p /opt/llamafile
sudo curl -L -o /opt/llamafile/model.llamafile \
  https://huggingface.co/mozilla-ai/llamafile_0.10/resolve/main/Qwen3.5-0.8B-Q8_0.llamafile
sudo chown root:root /opt/llamafile/model.llamafile
sudo chmod 755 /opt/llamafile/model.llamafile
```

Sprawdzenie opcji serwera w tej wersji:

```bash
/opt/llamafile/model.llamafile --help | grep -Ei 'server|host|port|browser|ngl|ctx'
```
W wersji 0.10 nie ma opcji `--nobrowser`, dlatego nie występuje w pliku jednostki.

## 3. Plik jednostki systemd

Plik `/etc/systemd/system/llamafile.service`:

```ini
[Unit]
Description=Llamafile LLM server (host-only)
After=network-online.target
Wants=network-online.target

[Service]
User=llamafile
Group=llamafile
WorkingDirectory=/opt/llamafile
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --host 192.168.128.2 --port 8080 -c 2048 -ngl 0
Environment=HOME=/var/lib/llamafile
StateDirectory=llamafile
Restart=always
RestartSec=3
NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

- **`/bin/sh`:** bezpośrednie uruchomienie przez systemd kończyło się błędem `status=203/EXEC`.
- **`Restart=always`:** `kill` wysyła SIGTERM, a przy `Restart=on-failure` systemd nie uruchomiłby usługi ponownie.
- **`--host 192.168.128.2`:** serwer nasłuchuje tylko na adresie host-only.

Uruchomienie:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now llamafile
```

## 4. Firewall

Reguła dla portu 8080 tylko na interfejsie host-only:

```bash
sudo ufw allow in on enp0s2 to any port 8080 proto tcp
sudo ufw status verbose
```

Bez tej reguły połączenie kończyło się timeoutem.

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp on enp0s2           ALLOW IN    Anywhere                  
22/tcp on enp0s3           ALLOW IN    Anywhere                  
8080/tcp on enp0s2         ALLOW IN    Anywhere                  
22/tcp (v6) on enp0s2      ALLOW IN    Anywhere (v6)             
22/tcp (v6) on enp0s3      ALLOW IN    Anywhere (v6)             
8080/tcp (v6) on enp0s2    ALLOW IN    Anywhere (v6)  
```

## 5. Weryfikacja

Status usługi:

```bash
systemctl status llamafile --no-pager
```

```
llamafile.service - Llamafile LLM server (host-only)
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-26 12:43:50 UTC; 48min ago
 Invocation: 9a93fc0fe7324db2b4552bd48e4e4f7c
   Main PID: 2164 (.ape-1.10)
      Tasks: 19 (limit: 948)
     Memory: 1019.9M (peak: 1.2G, swap: 335M, swap peak: 335.3M)
        CPU: 5.075s
     CGroup: /system.slice/llamafile.service
             └─2164 /var/lib/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --host 192.168.128.2 --port 8080 -c 2048 -ngl 0

paź 26 12:43:51 aleksander7 sh[2164]: 0.01.277.549 I srv  llama_server: listening on http://192.168.128.2:8080
paź 26 12:43:51 aleksander7 sh[2164]: 0.01.277.580 W srv  llama_server: NOTICE: server default port will be changed to :9931 in a future release
paź 26 12:43:51 aleksander7 sh[2164]: 0.01.277.603 W srv  llama_server:         ref: https://github.com/ggml-org/llama.cpp/pull/26508
paź 26 12:52:18 aleksander7 sh[2164]: 8.28.748.617 I slot get_availabl: id  3 | task -1 | selected slot by LRU, t_last = -1
paź 26 12:52:18 aleksander7 sh[2164]: 8.28.749.189 I slot launch_slot_: id  3 | task 0 | processing task, is_child = 0
paź 26 12:52:19 aleksander7 sh[2164]: 8.29.234.206 I slot print_timing: id  3 | task 0 | prompt eval time =     182.51 ms /    21 toke…per second)
paź 26 12:52:19 aleksander7 sh[2164]: 8.29.234.210 I slot print_timing: id  3 | task 0 |        eval time =     301.77 ms /    20 toke…per second)
paź 26 12:52:19 aleksander7 sh[2164]: 8.29.234.210 I slot print_timing: id  3 | task 0 |       total time =     484.28 ms /    41 tokens
paź 26 12:52:19 aleksander7 sh[2164]: 8.29.234.211 I slot print_timing: id  3 | task 0 |    graphs reused =         19
paź 26 12:52:19 aleksander7 sh[2164]: 8.29.236.069 I slot      release: id  3 | task 0 | stop processing: n_tokens = 40, truncated = 0
Hint: Some lines were ellipsized, use -l to show in full.
```
Nasłuch:

```bash
ss -tlnp | grep 8080
```

```
LISTEN 0      128    192.168.128.2:8080      0.0.0.0:*  
```

Zapytanie z gospodarza:

```bash
curl -v http://192.168.128.2:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Przedstaw się w jednym zdaniu."}],"max_tokens":60}'
```

```
*   Trying 192.168.128.2:8080...
* Connected to 192.168.128.2 (192.168.128.2) port 8080
POST /v1/chat/completions HTTP/1.1
Host: 192.168.128.2:8080
User-Agent: curl/8.7.1
Accept: */*
Content-Type: application/json
Content-Length: 90
upload completely sent off: 90 bytes < HTTP/1.1 200 OK < Server: llama.cpp < Access-Control-Allow-Origin: < Content-Type: application/json; charset=utf-8 < Content-Length: 789 < Keep-Alive: timeout=5, max=100 <
Connection #0 to host 192.168.128.2 left intact {"choices":[{"finish_reason":"stop","index":0,"message":{"role":"assistant","content":"Przedstaw się w jednym zdaniu to prośba do napisania jednej, bezpośredniej odpowiedzi na konkretny temat, zamiast długich rozpraszających tekstów."}}],"created":1793021909,"model":"/zip/Qwen3.5-0.8B-Q8_0.gguf","system_fingerprint":"b1789468725-0177dcc73","object":"chat.completion","usage":{"completion_tokens":39,"prompt_tokens":21,"total_tokens":60,"prompt_tokens_details":{"cached_tokens":17}},"id":"chatcmpl-P8oc4lJ6ckkVJswbvp8EXsiUSHl6HsFf","timings":{"cache_n":17,"prompt_n":4,"prompt_ms":57.587,"prompt_per_token_ms":14.39675,"prompt_per_second":69.4601212079115,"predicted_n":39,"predicted_ms":576.517,"predicted_per_token_ms":15.171500000000002,"predicted_per_second":65.91306067297235}}
```
Test pozostałych adresów maszyny z gospodarza:

```bash
curl --max-time 5 http://192.168.64.2:8080/v1/models
curl --max-time 5 http://10.16.0.105:8080/v1/models
```

```
curl: (28) Connection timed out after 5006 milliseconds
curl: (28) Connection timed out after 5007 milliseconds
```

Zużycie pamięci:

```bash
systemctl status llamafile --no-pager
```

```
Memory: 1G (peak: 1.2G, swap: 335M, swap peak: 335.3M)
```

## 6. Test kill i restart

Zabicie procesu, systemd ma go uruchomić ponownie:

```bash
PID1=$(systemctl show -p MainPID --value llamafile)
sudo kill $PID1
sleep 5
PID2=$(systemctl show -p MainPID --value llamafile)
echo "Przed: $PID1, po: $PID2"
```

```
Przed: 2164, po: 3524
```
Restart maszyny:

```bash
sudo reboot
```

Po ponownym połączeniu z gospodarza przez SSH:

```bash
systemctl status llamafile --no-pager -1
```
```
llamafile.service - Llamafile LLM server (host-only)
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-26 13:49:42 UTC; 5min ago
 Invocation: b9e9ceea16c148ff99dc2e9d68bac67e
   Main PID: 1258 (.ape-1.10)
      Tasks: 16 (limit: 948)
     Memory: 1.1G (peak: 1.2G, swap: 126.6M, swap peak: 126.7M)
        CPU: 2.136s
     CGroup: /system.slice/llamafile.service
             └─1258 /var/lib/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --host 192.168.128.2 --port 8080 -c 2048 -ngl 0

paź 26 13:49:43 aleksander7 sh[1258]: 0.00.439.332 I cmn          init: llama threadpool init, n_threads = 4
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.521.688 W load_hparams: Qwen-VL models require at minimum 1024 image tokens to function correctly on grounding tasks
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.521.696 W load_hparams: if you encounter problems with accuracy, try adding --image-min-tokens 1024
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.521.697 W load_hparams: more info: https://github.com/ggml-org/llama.cpp/issues/16842
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.682.358 I srv    load_model: loaded multimodal model, '/zip/d312c4d02fd46eea7a16e4f3bbb58840e6222209322ca1e33ca03247ad8935d6.gguf'
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.775.022 I srv    load_model: initializing, n_slots = 4, n_ctx_slot = 2048, kv_unified = 'true'
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.837.319 I srv  llama_server: model loaded
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.837.415 I srv  llama_server: listening on http://192.168.128.2:8080
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.837.457 W srv  llama_server: NOTICE: server default port will be changed to :9931 in a future release
paź 26 13:49:43 aleksander7 sh[1258]: 0.00.837.509 W srv  llama_server:         ref: https://github.com/ggml-org/llama.cpp/pull/26508
```


