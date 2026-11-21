# Ascend Inference Environment · Linux Command Runbook

Practice machine: 8×910B3 64GB | CANN 8.5.1 | vLLM / vLLM-Ascend 0.18.0 | Python 3.11.14 | AISBench 3.0.20251103

Conventions:

- Terminal A runs the inference service; Terminal B sends requests and runs benchmarks. All `docker exec` commands are typed on the host.
- Working directory `/tmp/exam_submit/work`; submission directory `/tmp/exam_submit/vllm_ascend`.
- The `export`s at the start of each block apply only to that terminal. Re-run them in a new terminal.
- When switching scenarios, make sure the previous round's processes are fully stopped before starting a new service.

---

## 1. Version and runtime environment checks

```bash
docker exec -it exam-q1q2 bash
mkdir -p /tmp/exam_submit/work/q1
cd /tmp/exam_submit/work/q1
```

```bash
cat /usr/local/Ascend/driver/version.info
python3 -m pip show vllm vllm-ascend torch torch-npu | grep -E '^(Name|Version)'
cat /cann/aarch64-linux/ascend_toolkit_install.info
python3 --version
printenv ASCEND_HOME_PATH ASCEND_OPP_PATH
python3 -c "import torch,torch_npu"
```

If the last command fails with a `torch_npu` import error, CANN and torch_npu are mismatched — install 8.5.1 as described in the next section.

## 2. Install CANN 8.5.1 and switch to it

```bash
cd /tmp/exam_submit/work/q1

# keep a backup of the existing install registry
cp -a /etc/ascend_install.info ./ascend_install.info.bak

# check installer usage first to confirm the flags
bash /mnt/share/spare_versions/Ascend-cann-toolkit_8.5.1_linux-aarch64.run --help
bash /mnt/share/spare_versions/Ascend-cann-910b-ops_8.5.1_linux-aarch64.run --help
```

Install the toolkit and the 910B operator package into the same prefix, `/opt/exam-cann851`:

```bash
bash /mnt/share/spare_versions/Ascend-cann-toolkit_8.5.1_linux-aarch64.run \
  --full --install-path=/opt/exam-cann851 2>&1 | tee toolkit-install.log
grep -i "install success" toolkit-install.log

bash /mnt/share/spare_versions/Ascend-cann-910b-ops_8.5.1_linux-aarch64.run \
  --install --type=toolkit --install-path=/opt/exam-cann851 2>&1 | tee ops-install.log
grep -i "install success" ops-install.log
```

Switch to the new version and confirm the NPU is usable:

```bash
export ASCEND_HOME_PATH=/opt/exam-cann851/cann-8.5.1
source /opt/exam-cann851/cann-8.5.1/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

printenv ASCEND_HOME_PATH ASCEND_OPP_PATH
cat /opt/exam-cann851/cann-8.5.1/opp/version.info
python3 -c "import torch,torch_npu; print(torch.npu.is_available())"
```

`opp/version.info` showing 8.5.1 and `is_available()` returning `True` means the switch took effect. The version registry under `/cann` is left untouched; the path you just `source`d is what counts.

## 3. Start the Qwen3-8B service and verify the API

**Terminal A**

```bash
docker exec -it exam-q1q2 bash
export ASCEND_HOME_PATH=/opt/exam-cann851/cann-8.5.1
source /opt/exam-cann851/cann-8.5.1/set_env.sh
export ASCEND_RT_VISIBLE_DEVICES=0

vllm serve /models/Qwen3-8B \
  --served-model-name qwen3-8b --host 127.0.0.1 --port 8101 \
  --tensor-parallel-size 1 --max-model-len 4096 \
  --gpu-memory-utilization 0.80 --enforce-eager \
  2>&1 | tee /tmp/exam_submit/work/q1/service.log
```

Once `Application startup complete` appears in the log, leave Terminal A alone.

**Terminal B**

```bash
docker exec -it exam-q1q2 bash
cd /tmp/exam_submit/work/q1

tail -n 4 service.log | tee startup-last4.log
curl -f -i http://127.0.0.1:8101/health

curl --fail-with-body -sS http://127.0.0.1:8101/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-8b","messages":[{"role":"user","content":"hello"}],"temperature":0,"max_tokens":64,"chat_template_kwargs":{"enable_thinking":false}}' \
  > response.json

jq '.choices[0] | {content:.message.content,finish_reason}' response.json
```

A 200 from `/health` and a non-empty `content` from `jq` mean the service is healthy.

## 4. Verify weight-file integrity

```bash
docker exec -it exam-q1q2 bash
mkdir -p /tmp/exam_submit/work/q2
cd /tmp/exam_submit/work/q2
```

List both directories first to check for symlinks and compare file sizes:

```bash
find /models/Qwen3-8B -printf '%y %p %s %l\n'
find /mnt/share/qwen3-8B -printf '%y %p %s %l\n'
```

Then compute SHA256 per file and compare:

```bash
cd /models/Qwen3-8B
find . -type f -print0 | sort -z | xargs -0 sha256sum > /tmp/exam_submit/work/q2/reference.sha256

cd /mnt/share/qwen3-8B
find . -type f -print0 | sort -z | xargs -0 sha256sum > /tmp/exam_submit/work/q2/question.sha256

cd /tmp/exam_submit/work/q2
wc -l reference.sha256 question.sha256
diff -u reference.sha256 question.sha256
diff -u /models/Qwen3-8B/config.json /mnt/share/qwen3-8B/config.json
```

Any `diff` output means the trees differ (exit code 1 is expected).

Locate the anomalous parameters and confirm the corrupted shard:

```bash
python3 -c "import json; a=json.load(open('/models/Qwen3-8B/config.json')); b=json.load(open('/mnt/share/qwen3-8B/config.json')); print([(k,a.get(k),b.get(k)) for k in sorted(set(a)|set(b)) if a.get(k)!=b.get(k)])"

python3 -c "from safetensors import safe_open; f=safe_open('/mnt/share/qwen3-8B/model-00001-of-00005.safetensors',framework='pt',device='cpu'); print(len(f.keys()))"

python3 -c "from safetensors import safe_open; f=safe_open('/mnt/share/qwen3-8B/model-00003-of-00005.safetensors',framework='pt',device='cpu'); print(len(f.keys()))"
```

A shard that fails to open (header invalid UTF-8) is corrupt, and that directory cannot be used to serve directly.

## 5. Deploy the Qwen3-30B deterministic service

**Terminal A**

```bash
docker exec -it exam-q3 bash
mkdir -p /tmp/exam_submit/work/q3
cd /tmp/exam_submit/work/q3
vi start_script.sh
```

Contents of `start_script.sh`:

```bash
#!/bin/bash
export ASCEND_HOME_PATH=/usr/local/Ascend/cann-8.5.1
source /usr/local/Ascend/cann-8.5.1/set_env.sh
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3
export LCCL_DETERMINISTIC=true
export HCCL_DETERMINISTIC=true
export ACL_OP_DETERMINISTIC=true
export ATB_MATMUL_SHUFFLE_K_ENABLE=0

exec vllm serve /models/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3-30b \
  --host 127.0.0.1 --port 8000 --trust-remote-code \
  --max-model-len 32768 --gpu-memory-utilization 0.90 \
  --max-num-seqs 16 --tensor-parallel-size 4 \
  --seed 0 --enforce-eager
```

```bash
chmod +x start_script.sh
bash start_script.sh 2>&1 | tee server.log
```

Determinism rests on these switches: the three `*_DETERMINISTIC=true` flags for LCCL/HCCL/ACL, `ATB_MATMUL_SHUFFLE_K_ENABLE=0`, `--seed 0`, and `--enforce-eager`.

**Terminal B**

```bash
docker exec -it exam-q3 bash
cd /tmp/exam_submit/work/q3
curl -f -i http://127.0.0.1:8000/health
vi curl_script.sh
```

Contents of `curl_script.sh`:

```bash
#!/bin/bash
curl --fail-with-body -sS --connect-timeout 10 --max-time 300 \
  http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3-30b",
    "messages": [{"role": "user", "content": "请介绍昇腾开发者工作级认证项目"}],
    "temperature": 0,
    "top_k": 10,
    "top_p": 1,
    "seed": 0,
    "repetition_penalty": 1.0,
    "max_tokens": 50,
    "stream": false
  }'
```

Send it three times and confirm the answers are byte-identical:

```bash
chmod +x curl_script.sh
./curl_script.sh > response-1.json
./curl_script.sh > response-2.json
./curl_script.sh > response-3.json

jq '.usage.completion_tokens' response-1.json response-2.json response-3.json

jq -j '.choices[0].message.content' response-1.json > answer-1.txt
jq -j '.choices[0].message.content' response-2.json > answer-2.txt
jq -j '.choices[0].message.content' response-3.json > answer-3.txt

cmp answer-1.txt answer-2.txt
cmp answer-1.txt answer-3.txt
cp -i answer-1.txt answer_50token.txt
```

All three runs reporting `completion_tokens` = 50 and both `cmp` calls printing nothing means determinism and the 50-token output are both correct.

## 6. GSM8K accuracy evaluation

```bash
docker exec -it exam-q3 bash
cd /tmp/exam_submit/work/q3

AIS=/usr/local/python3.11.14/lib/python3.11/site-packages/ais_bench

cat $AIS/benchmark/configs/models/vllm_api/vllm_api_general_chat.py
cat $AIS/datasets/gsm8k/test.jsonl
cp -i $AIS/benchmark/configs/models/vllm_api/vllm_api_general_chat.py aisbench_config.py
```

Check the four items in the template: `model=qwen3-30b`, endpoint `127.0.0.1:8000`, `abbr=vllm_api_general_chat`, and a 20-line data file.

Run the evaluation:

```bash
ais_bench --models vllm_api_general_chat \
  --datasets gsm8k_gen_4_shot_cot_str \
  --summarizer example \
  --mode all --debug \
  --work-dir /tmp/exam_submit/work/q3-gsm8k-01
```

Note the `Current exp folder` path printed at the end, then extract the score:

```bash
read -r -p 'Paste the full Current exp folder path: ' Q3_EXP
jq '.accuracy / 100' "$Q3_EXP/results/vllm_api_general_chat/gsm8k.json" > accuracy_score.txt
cat accuracy_score.txt
```

`accuracy_score.txt` holds a decimal between 0 and 1 (e.g. 0.9).

## 7. Performance-test preparation (config templates and backups)

```bash
docker exec -it exam-q4 bash
mkdir -p /tmp/exam_submit/work/q4
cd /tmp/exam_submit/work/q4

export WORK=/tmp/exam_submit/work/q4
export AIS_ROOT=/usr/local/python3.11.14/lib/python3.11/site-packages/ais_bench
export MODEL_CFG=$AIS_ROOT/benchmark/configs/models/vllm_api/vllm_api_general_stream.py
export SYNTH_CFG=$AIS_ROOT/datasets/synthetic/synthetic_config.py
```

Those `export`s are just path aliases for the `vi` / `cat` / `cp` calls below — **ais_bench never reads them**. `--models` and `--datasets` take config *names* (`vllm_api_general_stream`, `synthetic_gen`), and the tool resolves names recursively by filename under `<--config-dir>/models`, `<--config-dir>/datasets` and the package default `configs/`; it does **not** accept file paths. If you would rather not memorize the paths, have the tool print them:

```bash
ais_bench -s --models vllm_api_general_stream --datasets synthetic_gen --summarizer example
```

`-s/--search` only prints the absolute paths of the matched config files; it does not run any task.

```bash
cat "$MODEL_CFG"
cat "$SYNTH_CFG"
cat "$AIS_ROOT/benchmark/configs/datasets/synthetic/synthetic_gen.py"
ais_bench --help
```

The two files are different in nature and are handled differently:

- The model config `vllm_api_general_stream.py` lives under `configs/models`, so the installed file need not be touched: copy it into your own directory, point `--config-dir <dir>` at that directory (keep the `models/`, `datasets/`, `summarizers/` subdirectories), and still call it by name. If the same name exists in both places the tool warns and prefers the one under `--config-dir`. This runbook takes the shortcut of editing the original file and restoring it afterwards.
- `synthetic_config.py` is not under `configs`, so `--config-dir` cannot reach it: `SyntheticDataset` hard-codes `from ais_bench.datasets.synthetic.synthetic_config import synthetic_config`, and the input/output lengths and `RequestCount` come only from that copy inside the package. So this file must be edited in place; the backup below and the restore in section 14 are not optional.

Save an untouched copy of both preinstalled configs first. Every round then edits the original files, and everything is restored after the last run (see section 14):

```bash
cp -a "$MODEL_CFG" "$WORK/vllm_api_general_stream.py.orig"
cp -a "$SYNTH_CFG" "$WORK/synthetic_config.py.orig"
ls -l "$WORK"/*.orig
```

The model side uses the `/v1/completions` streaming API under the config name `vllm_api_general_stream`; the dataset side uses `synthetic_gen`. Length and concurrency are controlled by the configs; `--num-prompts` only sets how many requests are sent.

## 8. Task 4: parameter overview

Both answers share one service shape: TP8 + expert parallelism, the `mp` backend, 15-token Suffix speculative decoding, `FULL_DECODE_ONLY` graph mode, `weight_nz_mode=2`, prefix caching disabled, FlashComm1 enabled. Only the batching and VRAM budgets differ.

| Parameter | 4.2 · 32000/1024 · 16 concurrent | 4.3 · 64000/1024 · 8 concurrent |
| --- | --- | --- |
| 64 GB card `--max-model-len` | 40960 | 131072 |
| 64 GB card `--gpu-memory-utilization` | 0.88 | 0.92 |
| 64 GB card `--max-num-batched-tokens` | 98304 | 65536 |
| 64 GB card `--long-prefill-token-threshold` | 6144 | 8192 |
| 32 GB card `--max-model-len` | 36864 | 65536 |
| 32 GB card `--gpu-memory-utilization` | 0.90 | 0.90 |
| 32 GB card `--max-num-batched-tokens` | 32768 | 32768 |
| 32 GB card `--long-prefill-token-threshold` | 2048 | 4096 |
| Chunked prefill on 32 GB cards | `--enable-chunked-prefill --block-size 128` | same as left |
| Reference result (average of two launches on 64 GB) | total output throughput **881.03 token/s**, avg TPOT **3.54 ms** | **338.33 token/s**, **4.80 ms** |

The 64 GB column is the measured optimum; the 32 GB column is a first-try estimate derived from the VRAM budget — real performance is whatever the on-site benchmark says.

The VRAM math for 32 GB cards (TP8, DCP1, BF16 KV, one KV head kept per card): each token costs `48 × 2 × 1 × 128 × 2 = 24576` bytes. With all requests resident at once, 4.2 needs about `530432` KV tokens ≈ **12.14 GiB** and 4.3 about `521216` ≈ **11.93 GiB**, while the nominal 32 GiB × 0.90 budget leaves only around ten GiB after other overheads. So both the batched-token budget and the long-prefill threshold must come down, and chunked prefill lets long inputs arrive in several steps. In the end, trust `GPU KV cache size` in the startup log and the preemption counters during the benchmark.

## 9. Launch the benchmark service (Terminal A)

```bash
docker exec -it exam-q4 bash
mkdir -p /tmp/exam_submit/work/q4/{q4_2_64g,q4_3_64g,q4_2_32g,q4_3_32g}

source /usr/local/Ascend/cann-8.5.1/set_env.sh
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
export PYTHONHASHSEED=0
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_OP_EXPANSION_MODE=AIV
export HCCL_BUFFSIZE=1024
export OMP_NUM_THREADS=1
export TASK_QUEUE_ENABLE=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1
```

Both scenarios share the eight cards and `127.0.0.1:18081`, one at a time. If the target configuration is already running, skip straight to section 10.

### 4.2 · 64 GB cards

```bash
CFG=/tmp/exam_submit/work/q4/q4_2_64g
vi "$CFG/opt_script_q4_2.sh"
```

```bash
#!/bin/bash
source /usr/local/Ascend/cann-8.5.1/set_env.sh
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
export PYTHONHASHSEED=0
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_OP_EXPANSION_MODE=AIV
export HCCL_BUFFSIZE=1024
export OMP_NUM_THREADS=1
export TASK_QUEUE_ENABLE=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

vllm serve /models/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3 \
  --host 127.0.0.1 --port 18081 \
  --trust-remote-code \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --distributed-executor-backend mp \
  --max-model-len 40960 --max-num-seqs 16 \
  --gpu-memory-utilization 0.88 \
  --max-num-batched-tokens 98304 --long-prefill-token-threshold 6144 \
  --no-enable-prefix-caching \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY","cudagraph_capture_sizes":[16,32,64,128,256],"max_cudagraph_capture_size":256}' \
  --additional-config '{"weight_nz_mode":2}' \
  --speculative-config '{"method":"suffix","num_speculative_tokens":15,"suffix_decoding_max_tree_depth":32}'
```

```bash
chmod +x "$CFG/opt_script_q4_2.sh"
bash "$CFG/opt_script_q4_2.sh" 2>&1 | tee "$CFG/service-4.2.log"
```

The 16 sequence slots of TP8 are scheduled by a single engine, which exactly covers this scenario's 16 concurrent requests.

### 4.3 · 64 GB cards

```bash
CFG=/tmp/exam_submit/work/q4/q4_3_64g
vi "$CFG/opt_script_q4_3.sh"
```

```bash
#!/bin/bash
source /usr/local/Ascend/cann-8.5.1/set_env.sh
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
export PYTHONHASHSEED=0
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_OP_EXPANSION_MODE=AIV
export HCCL_BUFFSIZE=1024
export OMP_NUM_THREADS=1
export TASK_QUEUE_ENABLE=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

vllm serve /models/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3 \
  --host 127.0.0.1 --port 18081 \
  --trust-remote-code \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --distributed-executor-backend mp \
  --max-model-len 131072 --max-num-seqs 8 \
  --gpu-memory-utilization 0.92 \
  --max-num-batched-tokens 65536 --long-prefill-token-threshold 8192 \
  --no-enable-prefix-caching \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY","cudagraph_capture_sizes":[16,32,64,128],"max_cudagraph_capture_size":128}' \
  --additional-config '{"weight_nz_mode":2}' \
  --speculative-config '{"method":"suffix","num_speculative_tokens":15,"suffix_decoding_max_tree_depth":32}'
```

```bash
chmod +x "$CFG/opt_script_q4_3.sh"
bash "$CFG/opt_script_q4_3.sh" 2>&1 | tee "$CFG/service-4.3.log"
```

### Both scenarios on 32 GB cards

Use the same environment variables, replace `vllm serve` with the two commands below, and change the working directories to `q4_2_32g` and `q4_3_32g` accordingly.

4.2:

```bash
vllm serve /models/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3 \
  --host 127.0.0.1 --port 18081 \
  --trust-remote-code \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --distributed-executor-backend mp \
  --max-model-len 36864 --max-num-seqs 16 \
  --gpu-memory-utilization 0.90 \
  --enable-chunked-prefill --block-size 128 \
  --max-num-batched-tokens 32768 --long-prefill-token-threshold 2048 \
  --no-enable-prefix-caching \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY","cudagraph_capture_sizes":[16,32,64,128,256],"max_cudagraph_capture_size":256}' \
  --additional-config '{"weight_nz_mode":2}' \
  --speculative-config '{"method":"suffix","num_speculative_tokens":15,"suffix_decoding_max_tree_depth":32}'
```

4.3:

```bash
vllm serve /models/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3 \
  --host 127.0.0.1 --port 18081 \
  --trust-remote-code \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --distributed-executor-backend mp \
  --max-model-len 65536 --max-num-seqs 8 \
  --gpu-memory-utilization 0.90 \
  --enable-chunked-prefill --block-size 128 \
  --max-num-batched-tokens 32768 --long-prefill-token-threshold 4096 \
  --no-enable-prefix-caching \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY","cudagraph_capture_sizes":[16,32,64,128],"max_cudagraph_capture_size":128}' \
  --additional-config '{"weight_nz_mode":2}' \
  --speculative-config '{"method":"suffix","num_speculative_tokens":15,"suffix_decoding_max_tree_depth":32}'
```

### Health check and KV capacity check

```bash
curl -fsS -o /dev/null -w 'health=%{http_code}\n' http://127.0.0.1:18081/health
curl -fsS http://127.0.0.1:18081/v1/models
grep -E 'Available KV cache memory|GPU KV cache size|Maximum concurrency' "$CFG"/service-*.log | tail -n 3
```

Only when the health check returns 200, the model name is `qwen3`, and `Maximum concurrency` is at least this scenario's concurrency (16 or 8) should you start the benchmark.

## 10. Prepare the AISBench configs and run the benchmark (Terminal B)

```bash
docker exec -it exam-q4 bash
export CFG=/tmp/exam_submit/work/q4/q4_2_64g    # change to q4_3_64g for 4.3
export COUNT=16                                 # change to 8 for 4.3
export AIS=/usr/local/python3.11.14/lib/python3.11/site-packages/ais_bench
export MODEL_CFG=$AIS/benchmark/configs/models/vllm_api/vllm_api_general_stream.py
export SYNTH_CFG=$AIS/datasets/synthetic/synthetic_config.py
cd "$CFG"

curl -fsS -o /dev/null -w 'health=%{http_code}\n' http://127.0.0.1:18081/health
curl -fsS http://127.0.0.1:18081/v1/models
```

Store one config copy per scenario: `model.used.py` for the model side and `synthetic.used.py` for the dataset side. First the 4.2 versions:

```bash
vi "$CFG/model.used.py"
```

```python
from ais_bench.benchmark.models import VLLMCustomAPIStream

models = [dict(
    attr='service',
    type=VLLMCustomAPIStream,
    abbr='vllm-api-general-stream',
    path='/models/Qwen3-30B-A3B-Instruct',
    model='qwen3',
    request_rate=0,
    retry=0,
    host_ip='127.0.0.1',
    host_port=18081,
    max_out_len=1024,
    batch_size=16,
    trust_remote_code=True,
    generation_kwargs=dict(
        temperature=0, top_k=10, top_p=1, seed=0,
        repetition_penalty=1.0, ignore_eos=True,
    ),
    max_seq_len=40960,
)]
```

```bash
vi "$CFG/synthetic.used.py"
```

```python
synthetic_config = {
    'Type': 'string',
    'RequestCount': 16,
    'TrustRemoteCode': True,
    'StringConfig': {
        'Input': {'Method': 'uniform', 'Params': {'MinValue': 32000, 'MaxValue': 32000}},
        'Output': {'Method': 'uniform', 'Params': {'MinValue': 1024, 'MaxValue': 1024}},
    },
    'TokenIdConfig': {'RequestSize': 10},
}
```

For 4.3 only four things change in these two files: `batch_size` → `8`, `max_seq_len` → `131072`, `RequestCount` → `8`, and the Input `MinValue`/`MaxValue` → `64000`. On 32 GB cards `max_seq_len` becomes `36864` and `65536` respectively; everything else stays the same.

Overwrite the installed locations (backed up in section 7, restored in section 14):

```bash
cp "$CFG/model.used.py" "$MODEL_CFG"
cp "$CFG/synthetic.used.py" "$SYNTH_CFG"

grep -nE 'batch_size|max_seq_len|host_port' "$MODEL_CFG"
grep -nE 'RequestCount|MinValue|MaxValue' "$SYNTH_CFG"
```

The values printed by `grep` must match this scenario before you start:

```bash
RUN="$CFG/run-$(date +%Y%m%d_%H%M%S)"
mkdir -p "$RUN"

ais_bench --models vllm_api_general_stream --datasets synthetic_gen \
  --mode perf --num-prompts "$COUNT" \
  --work-dir "$RUN" 2>&1 | tee "$CFG/perf_text_q4_2.txt"
```

After the run, save this scenario's config copy under its submission filename (for 4.3 use `aisbench_config_q4_3.py` and `perf_text_q4_3.txt`):

```bash
cp "$CFG/model.used.py" "$CFG/aisbench_config_q4_2.py"
ls -l "$CFG"
```

## 11. Check results and validate every request

The result table is also printed to the terminal; read it back from `perf_text_q4_2.txt` afterwards:

```bash
grep -E 'Total Requests|Failed Requests|Success Requests|Concurrency|TPOT|Output Token Throughput' "$CFG/perf_text_q4_2.txt"
```

It only passes when the success count equals `$COUNT`, the failure count is 0, `Max Concurrency` equals `$COUNT`, and `TPOT / Average` ≤ 50 ms. Scenarios are compared on `Output Token Throughput / total`.

Note the `Current exp folder` printed in the terminal and validate lengths request by request:

```bash
read -r -p 'Paste the full Current exp folder path: ' EXP

jq '.requests.input_tokens_len | unique' "$EXP/performances/vllm-api-general-stream/syntheticdataset_details.json"
jq '.requests.generate_tokens_len | unique' "$EXP/performances/vllm-api-general-stream/syntheticdataset_details.json"

jq '{total:."Total Requests".total,success:."Success Requests".total,failed:."Failed Requests".total,max_concurrency:."Max Concurrency".total,throughput:."Output Token Throughput".total}' \
  "$EXP/performances/vllm-api-general-stream/syntheticdataset.json"

grep '^TPOT,total,' "$EXP/performances/vllm-api-general-stream/syntheticdataset.csv"
```

The `unique` lengths should be nothing but the target values (32000/1024 or 64000/1024).

Check the two service-side metrics as well:

```bash
curl -fsS http://127.0.0.1:18081/metrics | grep -E 'vllm:(num_preemptions_total|kv_cache_usage_perc)'
tail -n 80 "$CFG/service-4.2.log"
grep -nE 'ERROR|Traceback|OOM|preempt|timeout' "$CFG/service-4.2.log"
```

A steadily rising preemption counter means KV is short: cut the budget as described in section 12 and re-run, keeping the same request lengths and concurrency.

## 12. Adjustments when VRAM is insufficient

- **OOM during prefill or VRAM probing, or KV capacity below the reference in section 8**: drop `--max-num-batched-tokens` to `16384` and `--long-prefill-token-threshold` to `1024` for 4.2 or `2048` for 4.3. After restarting, recheck `GPU KV cache size` and re-run the benchmark with the same lengths and concurrency.
- **OOM while capturing ACLGraphs**: drop `num_speculative_tokens` to `7` and shrink the graph sizes at the same time — for 4.2 use `[8,16,32,64,128]` with a cap of `128`; for 4.3 use `[8,16,32,64]` with a cap of `64`. Then re-measure average TPOT and total output throughput.

## 13. Assemble and package the submission

```bash
mkdir -p /tmp/exam_submit/vllm_ascend/q1_q2 /tmp/exam_submit/vllm_ascend/q3 /tmp/exam_submit/vllm_ascend/q4
vi /tmp/exam_submit/vllm_ascend/q1_q2/answers.txt
wc -l /tmp/exam_submit/vllm_ascend/q1_q2/answers.txt
```

Write `answers.txt` line by line, in order: 6 version numbers (HDK, vLLM, vLLM-Ascend, CANN, Python, torch) → the faulty component name → the last three lines of the service startup log (four on a 64G machine) → the two corrupted shard filenames → the anomalous parameter name → the anomalous value → the expected value.

```bash
# five files for the accuracy task
cp -i /tmp/exam_submit/work/q3/start_script.sh \
      /tmp/exam_submit/work/q3/curl_script.sh \
      /tmp/exam_submit/work/q3/answer_50token.txt \
      /tmp/exam_submit/work/q3/aisbench_config.py \
      /tmp/exam_submit/work/q3/accuracy_score.txt \
      /tmp/exam_submit/vllm_ascend/q3/

# six files for the performance task
cp -i /tmp/exam_submit/work/q4/q4_2_64g/opt_script_q4_2.sh \
      /tmp/exam_submit/work/q4/q4_2_64g/aisbench_config_q4_2.py \
      /tmp/exam_submit/work/q4/q4_2_64g/perf_text_q4_2.txt \
      /tmp/exam_submit/work/q4/q4_3_64g/opt_script_q4_3.sh \
      /tmp/exam_submit/work/q4/q4_3_64g/aisbench_config_q4_3.py \
      /tmp/exam_submit/work/q4/q4_3_64g/perf_text_q4_3.txt \
      /tmp/exam_submit/vllm_ascend/q4/

chmod +x /tmp/exam_submit/vllm_ascend/q3/*.sh /tmp/exam_submit/vllm_ascend/q4/*.sh
```

On a 32 GB machine, replace `q4_2_64g` and `q4_3_64g` in the paths above with `q4_2_32g` and `q4_3_32g`.

Self-check everything before packaging:

```bash
find /tmp/exam_submit/vllm_ascend -type f | sort
find /tmp/exam_submit/vllm_ascend -type f | wc -l
wc -l /tmp/exam_submit/vllm_ascend/q1_q2/answers.txt
grep -n -- '--no-enable-prefix-caching' /tmp/exam_submit/vllm_ascend/q4/opt_script_q4_*.sh
grep -n 'num_speculative_tokens' /tmp/exam_submit/vllm_ascend/q4/opt_script_q4_*.sh
```

There should be 12 files (1 fill-in + 5 accuracy + 6 performance), `answers.txt` should have the required line count, and both startup scripts should show `--no-enable-prefix-caching` and `num_speculative_tokens`. Then package:

```bash
cd /tmp/exam_submit
zip -r /tmp/exam_submit/work/vllm_ascend.zip vllm_ascend
unzip -l /tmp/exam_submit/work/vllm_ascend.zip
```

The archive root is `vllm_ascend/`, and packaging does not require stopping the service.

## 14. Wrap-up: restore preinstalled configs, stop services

The install registry was backed up before the CANN installation in section 2. Check whether any installer rewrote it and restore if so:

```bash
diff /tmp/exam_submit/work/q1/ascend_install.info.bak /etc/ascend_install.info
cp -a /tmp/exam_submit/work/q1/ascend_install.info.bak /etc/ascend_install.info
```

Restore the two preinstalled configs backed up in section 7:

```bash
export WORK=/tmp/exam_submit/work/q4
export AIS_ROOT=/usr/local/python3.11.14/lib/python3.11/site-packages/ais_bench
export MODEL_CFG=$AIS_ROOT/benchmark/configs/models/vllm_api/vllm_api_general_stream.py
export SYNTH_CFG=$AIS_ROOT/datasets/synthetic/synthetic_config.py

cp -a "$WORK/vllm_api_general_stream.py.orig" "$MODEL_CFG"
cp -a "$WORK/synthetic_config.py.orig" "$SYNTH_CFG"

diff -q "$WORK/vllm_api_general_stream.py.orig" "$MODEL_CFG"
diff -q "$WORK/synthetic_config.py.orig" "$SYNTH_CFG"
```

Before switching scenarios, stop the previous service (check process ownership first so you do not kill someone else's run):

```bash
pgrep -af "vllm serve"
pkill -9 -f "vllm serve /models/Qwen3-30B-A3B-Instruct"
```

Confirm the port is released:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18081/health
```
