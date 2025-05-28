<script>
    import fastapi from "../lib/api";
    import Error from "../components/Error.svelte";
    import { link, push } from 'svelte-spa-router'
    import { is_login, username, keyword, answer_page, sort_by, desc } from "../lib/store"
    import { marked } from 'marked'
    import moment from 'moment/min/moment-with-locales'
    moment.locale('ko')
    export let params = {};
    let question_id = params.question_id;
    let question = { answers: [], voter:[], content: '' };
    let content = "";
    let error = { detail: [] };

    let answer_list = []
    let size = 10
    let total = 0

    $: total_page = Math.ceil(total/size)

    function get_question() {
        fastapi("get", "/api/question/detail/" + question_id, {}, (json) => {
            question = json;
        });
    }
    get_question();

    function post_answer(event) {
        event.preventDefault();
        let url = "/api/answer/create/" + question_id;
        let params = {
            content: content,
        };
        fastapi(
            'post',
            url,
            params,
            (json) => {
                content = '';
                error = { detail: [] };
                get_question();
            },
            (err_json) => {
                error = err_json;
            },
        );
    }

    function delete_question(_question_id) {
        if(window.confirm('정말로 삭제하시겠습니까?')) {
            let url = "/api/question/delete"
            let params = {
                question_id: _question_id
            }
            fastapi('delete', url, params, 
                (json) => {
                    push('/')
                },
                (err_json) => {
                    error = err_json
                }
            )
        }
    }
    function delete_answer(answer_id) {
        if(window.confirm('정말로 삭제하시겠습니까?')) {
            let url = "/api/answer/delete"
            let params = {
                answer_id: answer_id
            }
            fastapi('delete', url, params, 
                (json) => {
                    get_question()
                },
                (err_json) => {
                    error = err_json
                }
            )
        }
    }

    function vote_question(_question_id) {
        if(window.confirm('정말로 추천하시겠습니까?')) {
            let url = "/api/question/vote"
            let params = {
                question_id: _question_id
            }
            fastapi('post', url, params, 
                (json) => {
                    get_question()
                },
                (err_json) => {
                    error = err_json
                }
            )
        }
    }

    function vote_answer(answer_id) {
        if(window.confirm('정말로 추천하시겠습니까?')) {
            let url = "/api/answer/vote"
            let params = {
                answer_id: answer_id
            }
            fastapi('post', url, params, 
                (json) => {
                    get_question()
                },
                (err_json) => {
                    error = err_json
                }
            )
        }
    }

    function get_answer_list() {
        let params = {
            question_id: question_id,
            page: $answer_page,
            size: size,
            //keyword: $keyword,
            sort_by: $sort_by,
            desc: $desc
        }
        fastapi('get', '/api/answer/list', params, (json) => {
            answer_list = json.answer_list
            total = json.total
            //kw = $keyword
        },
        (err_json) =>{
            error = err_json
        })
    }

    $: $answer_page, $keyword, $sort_by, $desc, get_answer_list()

    function sort_voter() {
    $sort_by = 'voter_count';
    $desc = true;
}

</script>


<div class="container my-3">
    <!-- 질문 -->
    <h2 class="border-bottom py-2">{question.subject}</h2>
    <div class="card my-3">
        <div class="card-body">
            <div class="card-text" >{@html marked.parse(question.content)}</div>
            <div class="d-flex justify-content-end">
                {#if question.modify_date }
                <div class="badge bg-light text-dark p-2 text-start mx-3">
                    <div class="mb-2">modified at</div>
                    <div>{moment(question.modify_date).format("YYYY년 MM월 DD일 hh:mm a")}</div>
                </div>
                {/if}
                <div class="badge bg-light text-dark p-2 text-start">
                    <div class="mb-2">{ question.user ? question.user.username : ""}</div>
                    <div>{moment(question.create_date).format("YYYY년 MM월 DD일 hh:mm a")}</div>
                </div>
            </div>
            <div class="my-3">
                <button class="btn btn-sm btn-outline-secondary"
                    on:click="{vote_question(question.id)}"> 
                    추천
                    <span class="badge rounded-pill bg-success">{ question.voter.length }</span>
                </button>
                {#if question.user && $username === question.user.username }
                <a use:link href="/question-modify/{question.id}" 
                    class="btn btn-sm btn-outline-secondary">수정</a>
                    <button class="btn btn-sm btn-outline-secondary"
                    on:click={() => delete_question(question.id)}>삭제</button>
                {/if}
            </div>
        </div>
    </div>

    <button class="btn btn-secondary" on:click="{() => {
        push('/')
    }}">목록으로</button>

    <!-- 답변 목록 -->
    <!-- <h5 class="border-bottom my-3 py-2">{question.answers.length}개의 답변이 있습니다.</h5> -->
    <div class="row">
        <div class="col-6">
          <h5 class="border-bottom my-3 py-2 mb-0">{total}개의 답변이 있습니다.</h5>
        </div>
        <div class="col-6 d-flex justify-content-end align-items-center">
            <button class="btn mr-2 btn-outline-primary {$sort_by=== 'voter_count' ? 'active' : ''}"
                on:click={() => {sort_voter(),console.log()} }>추천순</button>
            <button class="btn btn-outline-primary {$sort_by === 'create_date' && $desc === true ? 'active' : ''}"
                on:click={() => {$sort_by = 'create_date',$desc = true}}>최신순</button>
            <button class="btn btn-outline-primary {$sort_by === 'create_date' && $desc === false ? 'active' : ''}"
                on:click="{() => ($sort_by = 'create_date', $desc = false)}">오래된순</button>
        </div>
    </div>
 
        {#each answer_list as answer (answer.id)}
    <div class="card my-3">
        <div class="card-body">
            <div class="card-text">{@html marked.parse(answer.content)}</div>
            <div class="d-flex justify-content-end">
                {#if answer.modify_date }
                <div class="badge bg-light text-dark p-2 text-start mx-3">
                    <div class="mb-2">(수정됨)</div>
                    <div>{moment(answer.modify_date).format("YYYY년 MM월 DD일 hh:mm a")}</div>
                </div>
                {/if}
                <div class="badge bg-light text-dark p-2 text-start">
                    <div class="mb-2">{answer.user ? answer.user.username : ""}</div>
                    <div>{moment(answer.create_date).format("YYYY년 MM월 DD일 hh:mm a")}</div>
                </div>
            </div>
            <div class="my-3">
                <button class="btn btn-sm btn-outline-secondary"
                    on:click="{() => vote_answer(answer.id)}">추천
                    <span class="badge rounded-pill bg-success">{answer.voter.length}</span>
                </button>
                {#if answer.user && $username === answer.user.username }
                <a use:link href="/answer-modify/{answer.id}" 
                    class="btn btn-sm btn-outline-secondary">수정</a>
                <button class="btn btn-sm btn-outline-secondary"
                    on:click={() => delete_answer(answer.id) }>삭제</button>
                {/if}
            </div>
        </div>
    </div>
    {/each}
    <!-- 답변 등록 -->
    <Error error={error} />
    <form method="post" class="my-3">
        <div class="mb-3">
            <textarea rows="10" bind:value={content} disabled={$is_login ? "" : "disabled"} class="form-control"></textarea>
        </div>
        <input type="submit" value="답변등록" class="btn btn-primary {$is_login ? '' : 'disabled'}" on:click="{post_answer}" />
    </form>
    <!-- 답변 목록 페이징 -->
<ul class="pagination justify-content-center my-4">

    <!-- 처음 -->
    <li class="page-item { $answer_page === 0 && 'disabled' }">
        <button class="page-link" on:click={() => $answer_page = 0}>
            처음
        </button>
    </li>

    <!-- 이전 -->
    <li class="page-item { $answer_page <= 0 && 'disabled' }">
        <button class="page-link" on:click={() => $answer_page--}>
            이전
        </button>
    </li>

    <!-- 페이지 번호들 -->
    {#each Array(total_page) as _, loop_page}
        {#if loop_page >= $answer_page - 5 && loop_page <= $answer_page + 5}
            <li class="page-item { loop_page === $answer_page && 'active' }">
                <button class="page-link" on:click={() => $answer_page = loop_page}>
                    {loop_page + 1}
                </button>
            </li>
        {/if}
    {/each}

    <!-- 다음 -->
    <li class="page-item { $answer_page >= total_page - 1 && 'disabled' }">
        <button class="page-link" on:click={() => $answer_page++}>
            다음
        </button>
    </li>

    <!-- 마지막 -->
    <li class="page-item { $answer_page === total_page - 1 && 'disabled' }">
        <button class="page-link" on:click={() => $answer_page = total_page - 1}>
            마지막 ({total_page})
        </button>
    </li>

</ul>
</div>