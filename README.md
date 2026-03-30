# fullstack-agent-harness

실제로 만든 구성:
참고로 음.. 그... Skill 과 Agents를 둘다 만든 이유는 Agents 는 자율성을 테스트해보려고, Skill로 한건 직접 실행해보면서 쓰려고 만들었는데 이래저래 해보니 상황에 따라 둘다 필요하더라구요. 첨에는 Agents만 만들려고 했었는데 중간에 단독으로 실행하는게 궁금해서 skill 로도 만듬.

Skills (/plan, /build, /evaluate)
→ 사용자가 단계별로 직접 제어하는 Manual 모드

Agents (planner, builder, evaluator AGENT.md)
→ 오케스트레이터가 자동으로 돌리는 Autonomous 모드

harness.sh
→ 터미널에서 전체 파이프라인을 자율 실행하는 스크립트
→ 각 에이전트가 별도 프로세스로 실행되어 완전한 컨텍스트 격리

Playwright MCP로 UI를 실제 브라우저에서 테스트하고,
Context7 MCP로 라이브러리 공식 문서를 확인하고,
GitHub MCP로 발견된 버그를 이슈로 자동 등록합니다.
