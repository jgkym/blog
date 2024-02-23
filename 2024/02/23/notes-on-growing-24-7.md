# Notes on Growing 24/7

## MISSON

- ❌ **READ** [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- ✅ **IMPLEMENT** [ColBERT](https://github.com/stanford-futuredata/ColBERT)



## MON

 [**#RAGatouille**](https://github.com/bclavie/RAGatouille). ColBERT를 쉽게 사용할 수 있도록 도와주는 라이브러리. 사용법은 크게 두 가지로 나뉨. 스크래치에서 부터 학습하는 방법과 저장된 모델이나 인덱스를 불러와서 쓰는 방법. RAGTrainer와 RAGPretrainedModel 클래스가 각각애 대응됨. 다만 아쉬웠던 건 API 설명이 불충분해서 사용법을 익히기 쉽지 않음. 인풋으로 사용하는 데이터가 어떤 형식이어야 되는지 알기 어려웠다 (저기 튜토리얼 한 구석에 설명이 있음). 이외에는 컴팩트하게 기능이 구성되어 있어 사용하기 수월했음.  



[**#ColBERT**](https://github.com/stanford-futuredata/ColBERT )관련 에러. RAGatouille로 인덱싱할 때 cuda 관련 에러가 발생한다. RAGatouille의 핵심 기능들은 ColBERT 라이브러리를 오버라이딩하고 있고, 발생하는 문제들은 ColBERT 함수 사용해서 발생됨. 

    **CUDA_HOME**

```bash
OSError: CUDA_HOME environment variable is not set. Please set it to your CUDA install root.
```

-> `import torch`를 쓰면 에러가 발생하지 않는 경우가 생김. 찾아보니 torch의 utils 중에는 cuda의 경로를 찾는 기능이 있기도

    **decompress_residuals_cpp.so**

```bash
ImportError: /data/ephemeral/home/.cache/torch_extensions/py39_cu117/decompress_residuals_cpp/decompress_residuals_cpp.so: cannot open shared object file: No such file or directory
```

무슨 수를 쓰더라도, 결국 이 에러로 귀결된다. 컴파일링 된 파일이 없다고 한다. 이 문제는 마찬가지로 cuda 경로 인식 문제로 판단됨. `nvcc`를 찾을 수 있어야 하지만, 현재 서버 환경에는 없다. `cudatoolkit-dev`를 설치해서 해결했다는 코멘트를 보았지만, 설치할 때 서버가 터져버림. cuda는 참 어렵다...



## TUE

**#Coarse vs. Fine granularity**.  



**#pip 캐시를 안전하게 지우기**

```bash
pip cache purge
```



**#설치한 package의 소스코드를 수정하고 싶다면?**

```python
# package의 설치 경로 확인
import your_package
print(your_package.__file__)

# or
print(your_package.__path__)
```



## WED

**#Colab으로 Indexing**. CUDA 문제로 ColBERT를 놓고 있었지만, 그동안 학습한게 아쉬워서 최후의 수단으로 코랩 환경을 사용. 여기에서는 인덱싱 작업이 정상적으로 수행됨. `faiss-gpu`를 사용하나 모델 학습만큼 gpu를 사용하지는 않는다. T4도 OK. 왜 그동안 이 생각을 안했을까. 에러와 일기토를 굳이 안했어도 되지 않았을까 싶음. 다양한 작업환경을 먼저 시도했더라면 더 좋았겠다. 메모. 무튼 인덱싱은 대개 30분 정도 걸림. 다만 인덱싱을 저장하고 불러와서 사용할 때에는 경로 인식 문제가 발생한다. 또한 search 과정에서도 CUDA 에러가 발생하는 것을 확인함. 따라서 인덱싱에서 부터  retrievals를 test dataset에 붙이 작업까지 하나의 파이프라인으로 만들어 외부에서 작업함. inference는 빠르다는 장점.  



**#감정을 기록하자**. 오늘 함께자라기에서 인상 깊었던 점. 곰곰히 생각해보니 학습을 하면서 그때 어떤 생각을 가졌는지 돌아보는게 좋을 것 같다는 생각이 들었다. 그래서 적어본다. nvcc 개새끼.



## THU

**#ColBERT 강화**. 현재 KLUE-MRC 데이터셋에서 3952개의 train examples를 사용해서 ColBERT를 사전학습 시키고 있음. Backbone을 klue/roberta-large를 써보았지만 결과가 좋지는 않았다. 언론진흥재단에서 공개한 kpfbert를 사용했을 때, 가장 좋았다. 관련된 문서를 잘 찾아오도록 더 많은 tranining examples를 준비함. 가장 유사한 데이터셋인 KorQuad v1을 합쳤고, 7만 여개의 질문과 문서 쌍을 준비함. 먼저 [intfloat/multilingual-e5-large](https://huggingface.co/intfloat/multilingual-e5-large)를 사용해서 Hard negative를 진행함. 이거 효과 굉장히 좋았다. 이전까지 한국어 기반 sentence transformers에 대해 회의적이었는데 생각이 바뀜. 실제로 triples 결과를 확인했는데, 육안으로 봤을 때도 positive와 negative passage가 매우 닮았다. 데이터 준비에서 부터 학습까지 총 8시간 정도 소요됐다. Indexing까지 약 1시간 추가. 성능은 매우 굿!



**#Hybrid retrieval**. ColBERT로 성능을 올렸지만 BM25 보다 조금 높은 수준이다. 그래서 DPR+BM25처럼 두 리트리버의 결과를 합쳐보기로 함.   

**어떻게 합쳐야 될까?** 

1. 먼저 BM25에서 관련 문서 300개를 추려내고 그 문서들을 ColBERT를 사용해서 rerank. 사실 이러면 결과가 BM25에 대체로 종속됨. 혹은 이도저도 아닌 상황이 되어버림. 

2. 그래서 두 리트리버에서 각각 100개의 문서를 뽑아서 합쳐봄. ColBERT rerank를 통해 200개의 문서에서 Top k를 추려냄. 또 이러면 ColBERT에 결과가 종속됨. ColBERT 입맛에 맞는 결과가 상위에 위치하기 때문에

3. 두 결과를 합치고, 스코어를 단순하게 Sorting하자. Score는 결국에 Prob이니깐 두 리트리버에서 모두 동일한 scale. 점수가 클수록 이 문서가 gold라는 확신은 같은 수준일 것. 따라서 top k에서 두 리트리버의 결과를 적절하게 다룰 수 있을 것.<u>실제로 리더보드 점수가 꽤 오름!</u>



**#Top k를 줄이자!** 10, 20, 30개로 k를 늘렸을 때 오히려 점수가 떨어짐. k가 커질수록 gold passage는 반드시 retrievals에 포함되겠지만, context가 길어질수록 reader는 오히려 헷갈림. 좋은 결과를 얻기 위해서 정확하고 컴팩트하게 문서를 불러와서 extraction을 하는게 좋을 것. retriever 성능이 좋다는 전제 하에 k를 줄여보았다. 5, 3, 2, 1로 k를 낮추었는데 k가 2일 때 성능이 최고.



## FRI

**#논문 구현**을 잘하고 싶다. 문제에 맞는 툴킷을 빠르게 익히는 것도 좋지만, 커스터마이징하는게 쉽지는 않다. 스크래치에서 부터 코드를 구현할 수 있으면 좋을 것 같다. 