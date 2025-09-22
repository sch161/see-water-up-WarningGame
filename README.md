import streamlit as st
import matplotlib.pyplot as plt
import random
import textwrap

st.set_page_config(page_title="Tofu(스트레스볼) 게임", layout="wide")

# ---------------------- 유틸리티 ----------------------
def reset_game():
    st.session_state.round = 0
    st.session_state.water_level = 0.0  # 0..1 (비율)
    st.session_state.ball_y = 0.8  # 스트레스볼 초기 높이 (비율)
    st.session_state.max_rounds = 10
    st.session_state.game_over = False
    st.session_state.log = []
    st.session_state.mission = ""
    st.session_state.mission_submitted = False
    st.session_state.mode = "single"


def next_mission():
    prompts = [
        "요즘 기분을 한 단어로 적어보세요 (예: 불안, 피곤, 신남)",
        "최근 스트레스를 준 한 가지를 적어보세요",
        "지금 당장 할 수 있는 작은 휴식 행동을 하나 적어보세요",
        "누군가에게 말하고 싶은 짧은 위로 메시지를 적어보세요",
        "마지막으로 웃었던 이유를 적어보세요"
    ]
    st.session_state.mission = random.choice(prompts)
    st.session_state.mission_submitted = False


def draw_scene(water_level, ball_y, show_axes=False):
    # 물과 스트레스볼을 비율로 그리는 간단한 시각화
    fig, ax = plt.subplots(figsize=(4, 6))
    ax.set_xlim(0, 1)
    ax.set_ylim(0, 1)

    # 바닥과 배경
    ax.add_patch(plt.Rectangle((0, 0), 1, 1, fill=True, color="#f6f5f3"))

    # 물 (파란색 직사각형) - 투명도 높음
    ax.add_patch(plt.Rectangle((0, 0), 1, water_level, color="#4aa3df", alpha=0.85))

    # 스트레스볼 (원)
    ball_x = 0.5
    ball_radius = 0.07
    circle = plt.Circle((ball_x, ball_y), ball_radius, color="#ff6666", ec="black", lw=1.0)
    ax.add_patch(circle)

    # 물과 볼 위치 표시선
    ax.hlines(water_level, 0, 1, linestyles="--", linewidth=1, colors="navy")
    ax.text(0.02, water_level + 0.02, f"물 높이: {water_level:.2f}", fontsize=9)
    ax.text(0.02, ball_y + 0.02, f"스트레스볼 높이: {ball_y:.2f}", fontsize=9)

    # 꾸밈
    if not show_axes:
        ax.axis('off')
    else:
        ax.set_xlabel('가로')
        ax.set_ylabel('세로')

    return fig


# ---------------------- 초기화 ----------------------
if 'round' not in st.session_state:
    reset_game()

# ---------------------- 사이드바 설정 ----------------------
with st.sidebar:
    st.title("Tofu 게임 (스트레스볼 버전)")
    st.markdown("**목표**: 스트레스(물)를 피해 스트레스볼을 안전한 곳으로 옮기세요.\n\n- 라운드가 지날수록 해수면(물 높이)이 올라갑니다.\n- 미션을 수행하여 팀(혹은 본인)이 스트레스를 관리하는 연습을 합니다.")

    if st.button("게임 초기화 (Reset)"):
        reset_game()
        st.experimental_rerun()

    st.write("---")
    st.write("### 설정")
    inc = st.slider("한 라운드에 물이 오르는 비율(증가량)", 0.03, 0.25, 0.08, step=0.01, help="값이 클수록 더 빨리 물이 찹니다.")
    st.session_state.water_increment = inc

    st.write("참여 모드")
    mode = st.radio("모드 선택", options=["single", "team"], format_func=lambda x: "개인(혼자)" if x=="single" else "팀(여러명)")
    st.session_state.mode = mode

    st.write("---")
    st.caption("개발자: 스트림릿 앱 예시 코드입니다. 수업·워크숍용으로 수정해 사용하세요.")

# ---------------------- 메인 UI ----------------------
st.title("🌊 Tofu 게임 — 스트레스볼과 해수면 상승 시뮬레이션")
col1, col2 = st.columns([2, 1])

with col1:
    fig = draw_scene(st.session_state.water_level, st.session_state.ball_y)
    st.pyplot(fig)

    st.write(f"**라운드:** {st.session_state.round} / {st.session_state.max_rounds}")
    st.write(f"**현재 물 높이:** {st.session_state.water_level:.2f} (비율, 0~1)")
    st.write(f"**스트레스볼 높이:** {st.session_state.ball_y:.2f}")

    if st.session_state.game_over:
        st.error("게임 종료: 스트레스(물)가 스트레스볼을 덮었습니다. 다시 시작하세요.")
    
with col2:
    st.subheader("라운드 진행")
    if st.session_state.round == 0 and not st.session_state.mission:
        if st.button("게임 시작"):
            st.session_state.round = 1
            next_mission()
            st.experimental_rerun()
    elif not st.session_state.game_over:
        st.markdown("**이번 라운드 미션**")
        st.write(textwrap.fill(st.session_state.mission, width=40))

        # 미션 제출 폼
        with st.form(key=f"mission_form_{st.session_state.round}"):
            user_reply = st.text_input("미션 응답 입력 (간단하게)")
            submitted = st.form_submit_button("제출")

            if submitted:
                st.session_state.mission_submitted = True
                # 성공 기준: 텍스트 비어있지 않음
                success = len(user_reply.strip()) > 0

                if success:
                    st.session_state.log.append((st.session_state.round, "성공", st.session_state.mission, user_reply))
                    # 성공 시: 스트레스볼을 살짝 끌어올리거나 물 상승량을 줄여 보상
                    st.session_state.ball_y = min(0.95, st.session_state.ball_y + 0.03)
                    st.session_state.water_level = max(0.0, st.session_state.water_level - 0.01)
                    st.success("미션 성공! 스트레스볼이 조금 올라갑니다. ✅")
                else:
                    st.session_state.log.append((st.session_state.round, "실패", st.session_state.mission, user_reply))
                    # 실패 시: 물이 더 많이 오른다
                    st.session_state.water_level = min(1.0, st.session_state.water_level + st.session_state.water_increment * 1.5)
                    st.warning("미션 미제출/빈 응답입니다. 물이 더 크게 올랐습니다. ⚠️")

                # 다음 라운드 준비
                if st.session_state.water_level >= st.session_state.ball_y:
                    st.session_state.game_over = True
                    st.session_state.log.append((st.session_state.round, "게임오버", "물에 잠김", ""))
                else:
                    st.session_state.round += 1
                    if st.session_state.round > st.session_state.max_rounds:
                        st.session_state.game_over = True
                        st.session_state.log.append((st.session_state.round, "완주", "최대 라운드 도달", ""))
                    else:
                        next_mission()

                st.experimental_rerun()

        st.write("---")
        if st.button("도전하지 않고 패스(물만 상승)"):
            # 패스하면 물만 상승
            st.session_state.log.append((st.session_state.round, "패스", st.session_state.mission, ""))
            st.session_state.water_level = min(1.0, st.session_state.water_level + st.session_state.water_increment)
            if st.session_state.water_level >= st.session_state.ball_y:
                st.session_state.game_over = True
                st.session_state.log.append((st.session_state.round, "게임오버", "물에 잠김", ""))
            else:
                st.session_state.round += 1
                if st.session_state.round > st.session_state.max_rounds:
                    st.session_state.game_over = True
                    st.session_state.log.append((st.session_state.round, "완주", "최대 라운드 도달", ""))
                else:
                    next_mission()
            st.experimental_rerun()

# ---------------------- 결과 로그 및 팁 ----------------------
st.write("---")
st.subheader("활동 로그")
if st.session_state.log:
    for r, status, mission, reply in st.session_state.log:
        st.write(f"라운드 {r}: {status} — 미션: {mission} — 응답: {reply}")
else:
    st.write("아직 진행 기록이 없습니다. 게임을 시작하세요.")

st.write("---")
st.subheader("진행자 팁 & 확장 아이디어")
st.markdown(
    """
- **교실/워크숍**: 팀 모드로 진행하면 한 사람이 미션을 수행할 때 다른 팀원이 마음을 지지하는 문장을 말해주는 규칙을 추가하세요.\n
- **시각적 연출**: 바닥에 파란 천을 깔고 실제로 높이를 매번 올리면 더 실감납니다.\n
- **짧은 대화**: 미션 제출 후, 참가자가 자발적으로 짧게 감정을 나누게 해서 토크 타임을 만드세요.\n
- **응급 신호 만들기**: 물이 너무 높아진 참가자는 사회자에게 신호를 보내고 휴식 시간을 갖습니다.\n
"""
)

st.write("\n---\n실행 방법: 터미널에서 `streamlit run tofu_game_streamlit.py` 를 실행하세요.")
