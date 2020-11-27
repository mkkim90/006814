# 자바와 JUnit을 활용한 실용주의 단위 테스트(길벗, 2019)

- 이 책의 모든 예제 코드는 자바 8을 기준으로 합니다.
- 일부 예제 코드는 일부러 오류가 발생하도록 작성된 것도 있습니다.


단위 테스스는 주의 깊게 사용했을 때 많은 이점을 얻을 수 있음
하지만 테스트 또한 유지 보수해야하는 또 다른 코드임

FIRST 원리를 따르면, 하기 문제점에 대해 대응할 수 있음 (개인적인 의견 아님 책에서 그랬음)

- 테스트를 사용하는 사람에게 어떤 정보도 주지 못하는 테스트
- 산발적으로 실패하는 테스트
- 어떤 가치도 증명하지 못하는 테스트
- 실행하는 데 오래 걸리는 테스트 
- 코드를 충분히 커버하지 못하는 테스트
- 구현과 강하게 결합되어 있는 테스트 따라서 작은 변화에도 다수의 테스트가 깨짐
- 수많은 설정 고리로 점프하는 난해한 테스트 


그렇다면 FIRST의 원리란 무엇인가 
하기와 같은 좋은 테스트 조건이다 

[F]ast : 빠른
[I]solated : 고립된
[R]epeatable : 반복 가능한
[S]elf-validating : 스스로 검증 가능한
[T]imely : 적시의


[F]IRST : 빠르다

테스트를 빠르게 유지해야됨 -> 근데 현실적으로 시스템이 커지면 단위 테스트도 실행하는데 점점 오래 걸림

EX ) 
하기 코드와 같이 QuestionController find 메소드(굵은 표시 부분)의 DB 호출 데이터와 상호작용하는 테스트는 느림

public class StatCompiler {
   static Question q1 = new BooleanQuestion("Tuition reimbursement?");
   static Question q2 = new BooleanQuestion("Relocation package?");

   class QuestionController {
      Question find(int id) {
         if (id == 1)
            return q1;
         else
            return q2;
      }
   }

   private QuestionController controller = new QuestionController();

   public Map<String, Map<Boolean, AtomicInteger>> responsesByQuestion(
         List<BooleanAnswer> answers) {
      Map<Integer, Map<Boolean, AtomicInteger>> responses = new HashMap<>();
      answers.stream().forEach(answer -> incrementHistogram(responses, answer));
      return convertHistogramIdsToText(responses);
   }

   private Map<String, Map<Boolean, AtomicInteger>> convertHistogramIdsToText(
         Map<Integer, Map<Boolean, AtomicInteger>> responses) {
      Map<String, Map<Boolean, AtomicInteger>> textResponses = new HashMap<>();
      responses.keySet().stream().forEach(id -> 
         textResponses.put(controller.find(id).getText(), responses.get(id)));
      return textResponses;
   }

   private void incrementHistogram(
         Map<Integer, Map<Boolean, AtomicInteger>> responses, 
         BooleanAnswer answer) {
      Map<Boolean, AtomicInteger> histogram = 
            getHistogram(responses, answer.getQuestionId());
      histogram.get(Boolean.valueOf(answer.getValue())).getAndIncrement();
   }

   private Map<Boolean, AtomicInteger> getHistogram(
         Map<Integer, Map<Boolean, AtomicInteger>> responses, int id) {
      Map<Boolean, AtomicInteger> histogram = null;
      if (responses.containsKey(id)) 
         histogram = responses.get(id);
      else {
         histogram = createNewHistogram();
         responses.put(id, histogram);
      }
      return histogram;
   }

   private Map<Boolean, AtomicInteger> createNewHistogram() {
      Map<Boolean, AtomicInteger> histogram;
      histogram = new HashMap<>();
      histogram.put(Boolean.FALSE, new AtomicInteger(0));
      histogram.put(Boolean.TRUE, new AtomicInteger(0));
      return histogram;
   }
}


questions 맴을 넘김으로써 테스트는 빨라짐

 public Map<String, Map<Boolean, AtomicInteger>> responsesByQuestion(
         List<BooleanAnswer> answers, Map<Integer, String> questions)  {
      Map<Integer, Map<Boolean, AtomicInteger>> responses = new HashMap<>();
      answers.stream().forEach(answer -> incrementHistogram(responses, answer));
      return convertHistogramIdsToText(responses, questions);
 }


private Map<String, Map<Boolean, AtomicInteger>> convertHistogramIdsToText(
         Map<Integer, Map<Boolean, AtomicInteger>> responses, Map<Integer, String> questions) {
      Map<String, Map<Boolean, AtomicInteger>> textResponses = new HashMap<>();
      responses.keySet().stream().forEach(id -> 
         textResponses.put(questions.get(id), responses.get(id)));
      return textResponses;
   }


DB 연결 없이 테스트 가능

 @Test
   public void test() {
      StatCompiler stats = new StatCompiler();
      
      List<BooleanAnswer> answers = new ArrayList<>();
      answers.add(new BooleanAnswer(1, true));
      answers.add(new BooleanAnswer(1, true));
      answers.add(new BooleanAnswer(1, true));
      answers.add(new BooleanAnswer(1, false));
      answers.add(new BooleanAnswer(2, true));
      answers.add(new BooleanAnswer(2, true));
      
      Map<String, Map<Boolean,AtomicInteger>> responses = stats.responsesByQuestion(answers);
      
      assertThat(responses.get("Tuition reimbursement?").get(Boolean.TRUE).get(), equalTo(3));
      assertThat(responses.get("Tuition reimbursement?").get(Boolean.FALSE).get(), equalTo(1));
      assertThat(responses.get("Relocation package?").get(Boolean.TRUE).get(), equalTo(2));
      assertThat(responses.get("Relocation package?").get(Boolean.FALSE).get(), equalTo(0));
   }



테스트 코드를 빠르게 동작하며 느린것에 의존하는 코드를 최소화-> 작성하기 쉬워짐




F[I]RST : 고립시킨다 

좋은 단윙 테스트는 검증하려는 작은 양의 코드에 집중
상호 작용하는 코드가 많을수록 문제가 발생할 소지가 늘어남

각 테스트가 작은 양의 동작에만 집중하면 테스트 코드를 집중적으로 독립적으로 유지하기 쉬움

객체 지향 클래스 설계의 단일 책임 원칙(SRP)에 따르면 클래스는 작고 단일한 목적을 가져야함


FI[R]ST : 좋은 테스트는 반복 가능해야 한다.

반복 가능한 테스트는 실행할 때마다 결과가 같아야 합니다. 따라서 반복 가능한 테스트를 만들려면
직접 통제할 수 없는 외부 환경에 있는 항목들과 격리시켜야함

각 테스트는 항상 동일한 결과를 만들어 내야함


FIR[S]T : 스스로 검증 가능하다

테스트는 스스로 검증 가능할 뿐만 아니라 준비할수 도 있어야 함
자동화해야됨


FIRS[T] : 적시에 사용한다



결론, 의존성 최소화 독립적인 객체지향적 프로그래밍이 테스트를 간소화한다. (개인의견)

참고 : 자바와 Junit을 활용한 실용주위 단위 테스트
참고 : https://github.com/gilbutITbook/006814/tree/master/iloveyouboss_16-branch-persistence-redesign 
