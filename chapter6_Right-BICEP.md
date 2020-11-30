# Right-BICEP은 무엇을 테스트할지에 대해 쉽게 선별
- *Right* :  결과가 올바른가?
- *B* : 경게조건은 맞는가?
- *I* : 역 관계를 검사할 수 있는가?
- *C* : 다른 수단을 활용하여 교차 검사할 수 있는가?
- *E* : 오류조건을 강제로 일어나게 할 수 있는가?
- *P* : 성능 조건은 기준에 부합하는가?


## Right 결과가 올바른가?

테스트 코드는 무엇보다도 먼저 기대한 결과를 산출하는지 검증할 수 있어야함 

## B 결계조건은 맞는가?

- 모호하고 일관성 없는 입력 값, 예를 들어 특수 문자가 포함된 파일이름

- 잘못된 양식의 데이터, 예를 들어 최상위 도메인이 빠진 이메일 주소 

- 수치적 오버플로를 일으키는 계산 

- 비거나 빠진 값. 예를 들어 0, 0.0 "" 혹은 NULL

- 이성적인 기댓값을 훨씬 벗어나는 값. 예를 들어 150세의 나이

- 교실의 당번표처럼 중복을 허용해서는 안되는 목록에 중복 값이 있는 경우

- 정렬이 안된 정렬 리스트 혹은 그 반대. 정렬 알고리즘에 이미 정렬된 입력 값을 넣는 경우나 정렬 알고리즘에 역순 데이터를 넣는 경우

- 시간 순이 맞지 않는 경우. 예를 들어 HTTP 서버가 OPTIONS 메서드의 결과를 POST 메서드보다 먼저 반환해야 하지만 그 후에 반환하는 경우

<hr/>


EX ) Null 인 경우 경계조건
```
@Test(expected=IllegalArgumentException.class)
   public void throwsExceptionWhenAddingNull() {
      collection.add(null);
   }
```

```
public void add(Scoreable scoreable) {
      if (scoreable == null) throw new IllegalArgumentException();
      scores.add(scoreable);
   }
```

EX ) 0으로 나누기 오류인 AtithmeticException이 발생할 수도 있음

```
@Test
   public void answersZeroWhenNoElementsAdded() {
      assertThat(collection.arithmeticMean(), equalTo(0));
   }
```

```
public int arithmeticMean() {
      if (scores.size() == 0) return 0;
      // ...
```

EX ) 큰 정수 입력을 다룬다면 숫자들의 합이 Integer.MAX_VALUE를 초과할 수 있음

```
@Test
   public void dealsWithIntegerOverflow() {
      collection.add(() -> Integer.MAX_VALUE); 
      collection.add(() -> 1); 
      
      assertThat(collection.arithmeticMean(), equalTo(1073741824));
   }
```

```
long total = scores.stream().mapToLong(Scoreable::getScore).sum();
      return (int)(total / scores.size());
```

## 경계 조건에서는 CORRECT를 기억하라 

- [C]onformance (준수) : 값이 기대한 양식을 준수하고 있는가?
- [O]rdering(순서) : 값의 집합이 적절하게 정렬되거나 정렬되지 않앗나?
- [R]ange(범위) : 이성적인 최솟값과 최댓값 안에 있는가? 
- [R]eference(참조) : 코드 자체에서 통제할 수 없는 어떤 외부 참조를 포함하고 있는가?
- [E]xistence(존재) : 값이 존재하는가 (널이 아니거나, 0이 아니거나, 집합에 존재하는가 등)?
- [C]ardinality(기수) : 정확히 충분한 값들이 있는가?
- [T]ime(절대적 혹은 상대적 시간) : 모든 것이 순서대로 일어나는가? 정확한 시간에? 정시에?



## Right-B[I]CEP : 역 관계를 검사할 수 있는가? 

때때로 논리적인 역 관계를 적용하여 행동을 검사할 수 있다. 

```
 public List<Answer> find(Predicate<Answer> pred) {
      return answers.values().stream()
            .filter(pred)
            .collect(Collectors.toList());
 }
 ```
 
 ```
   int[] ids(Collection<Answer> answers) {
      return answers.stream()
            .mapToInt(a -> a.getQuestion().getId()).toArray();
   }
``` 

```
@Test
   public void findsAnswersBasedOnPredicate() {
      profile.add(new Answer(new BooleanQuestion(1, "1"), Bool.FALSE));
      profile.add(new Answer(new PercentileQuestion(2, "2", new String[]{}), 0));
      profile.add(new Answer(new PercentileQuestion(3, "3", new String[]{}), 0));

      List<Answer> answers =
         profile.find(a->a.getQuestion().getClass() == PercentileQuestion.class);

      assertThat(ids(answers), equalTo(new int[] { 2, 3 }));
```

교차검사에는 프레디케이트의 보어를 찾는 것도 포함됨 
즉, PercentileQuestion 외의 답변들을 찾는 것  긍정사례답변과 역답변을 합치면 전체가 나와야함

```
List<Answer> answersComplement =
         profile.find(a->a.getQuestion().getClass() != PercentileQuestion.class);

      List<Answer> allAnswers = new ArrayList<Answer>();
      allAnswers.addAll(answersComplement);
      allAnswers.addAll(answers);

      assertThat(ids(allAnswers), equalTo(new int[] { 1, 2, 3 }));
   }
```

## Right-BI[C]EP : 다른 수단을 활용하여 교차 검사 할 수 있는가?

뉴턴의 제곱근 로직이 제곱근의 자바 라이브러리를 사용 하여 동일한 결과를 내는지 확인 가능 

```
 public class NewtonTest {
   static class Newton {
      private static final double TOLERANCE = 1E-16;

      public static double squareRoot(double n) {
         double approx = n;
         while (abs(approx - n / approx) > TOLERANCE * approx)
            approx = (n / approx + approx) / 2.0;
         return approx;
      }
   }
```

Math.sqrt() 메서드 
```
@Test
   public void squareRootVerifiedUsingLibrary() {
      assertThat(Newton.squareRoot(1969.0), 
            closeTo(Math.sqrt(1969.0), Newton.TOLERANCE));
   }
```

## Right-BIC[E]P : 오류 조건을 강제로 일어나게 할수 있는가? 

행복 경로의 반대편 역시 테스트할 필요가 있습니다. 다음의 시나리오들을 테스트할 수 있는 방법들에 대해 고려해볼 필요가 있음

   - 메모리가 가득찰 때
   - 디스크 공간이 가득 찰 때
   - 벽시계 시간에 관한 문제들(ex. 서버와 클라이언트 간 시간이 달라서 발생하는 문제들)
   - 네트워크 가용성 및 오류들
   - 시스템 로드
   - 제한된 색상 팔레트
   - 매우 높거나 낮은 비디오 해상도
   
   
*좋은 단위 테스트는 단지 코드에 존재하는 로직 전체에 대한 커버리지를 달성하는 것이 아닙니다. 때때로 뒷주머니에서 
작은 창의력을 꺼내는 노력이 필요합니다. 가장 끔찍한 결함들은 종종 전혀 예상하지 못한 곳에서 나옵니다. *

## Right-BICE[P] : 성능 조건은 기준에 부합하는가? 

모든 성능 최적화 시도는 실제 데이터로 해야하며 추측을 기반으로 해서는 안됨
이 예시는 어떤 코드가 특정 시간안에 실행되는 지 단언

```
private long run(int times, Runnable func) {
      long start = System.nanoTime();
      for (int i = 0; i < times; i++)
         func.run();
      long stop = System.nanoTime();
      return (stop - start) / 1000000;
   }
```

```
@Test
   public void findAnswers() {
      int dataSize = 5000;
      for (int i = 0; i < dataSize; i++)
         profile.add(new Answer(
               new BooleanQuestion(i, String.valueOf(i)), Bool.FALSE));
      profile.add(
         new Answer(
           new PercentileQuestion(
                 dataSize, String.valueOf(dataSize), new String[] {}), 0));

      int numberOfTimes = 1000;
      long elapsedMs = run(numberOfTimes,
         () -> profile.find(
               a -> a.getQuestion().getClass() == PercentileQuestion.class));

      assertTrue(elapsedMs < 1000);
   }
```

### 주의사항 
- 전형적인 코드 덩어리를 충분한 횟수만큼 실행하길 원할 것입니다. 이렇게 타이밍과 cpu 클록 주기에 관한 이슈를 제거 (성능을 측정할때는 출분한 개수를 반복해야
결과 수치가 튀지 않음. 몇번만 하면 할때마다 들쭉날쭉할 수 있음)
   
- 반복하는 코드 부분을 자바가 최적화하지 못하는지 확인해야 함
   
- 최적화되지 않은 테스트는 한번에 수 밀리초가 걸리는 일반적인 테스트 코드들보다 느림 -> 느린 테스트들은 빠른 것과 분리 
   
- 동일한 머신이라도 실행 시간은 시스템 로드처럼 잡다한 요소에 따라 달라질 수 있습니다. (성능을 테스트할때는 사전 조건을 단단하게 정의해 놓아야 반복성과 재현성을 보장)
   
   
단위 성능 측정을 잘 사용하는 방법은 *변경 사항을 만들때 기준점으로 활용하는 것*임
find() 메서드에 있는 자바 람다 기반의 해법이 최적화 되지 않는다고 가정한다면, 성능이 향상되는지 확인하기 위해 람다 기반을 좀더 전통적인 해법으로 교체 싶을 수 있음
최적화를 하기전에 먼저 기준점으로 단지 현재 경과 시간을 측정하는 성능 테스트를 작성 (몇번 실행해보고 평균을 계산)
코드를 변경하고 성능 테스트를 다시 실행하고 결과를 비교함-> 상대적인 개선량을 찾으면 됨

    

