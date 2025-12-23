<template>
  <div class="app">
    <header class="topbar">
      <div class="brand">
        <div class="brand-logo">KS</div>
        <div>
          <p class="brand-title">Keller Cyber</p>
          <p class="brand-subtitle">Inteligência em segurança digital</p>
        </div>
      </div>
      <nav class="nav">
        <a href="#posts">Posts</a>
        <a href="#insights">Insights</a>
        <a href="#labs">Labs</a>
        <button class="admin-toggle" @click="showAuth = true" aria-label="Admin login">
          •
        </button>
      </nav>
    </header>

    <main class="layout">
      <section class="sidebar" id="posts">
        <div class="section-header">
          <p class="eyebrow">Cybersecurity Blog</p>
          <h1>Foco total em proteção, análise e resposta</h1>
          <p class="muted">
            Postagens com relatórios, códigos, vídeos e embeds para você construir
            uma postura de segurança madura.
          </p>
        </div>

        <div v-if="error" class="alert">{{ error }}</div>
        <div v-else-if="loading" class="loading">Carregando posts...</div>

        <div class="post-list">
          <article
            v-for="post in posts"
            :key="post.id"
            class="post-card"
            :class="{ active: post.id === selectedPost?.id }"
            @click="selectPost(post)"
          >
            <div class="post-meta">
              <span>{{ formatDate(post.created_at) }}</span>
              <span class="pill">{{ post.category || 'Threat Intel' }}</span>
            </div>
            <h3>{{ post.title }}</h3>
            <p>{{ post.excerpt }}</p>
          </article>
        </div>
      </section>

      <section class="content" id="insights">
        <div v-if="selectedPost" class="post-detail">
          <div class="post-hero">
            <div>
              <p class="eyebrow">{{ selectedPost.category || 'Cyber Ops' }}</p>
              <h2>{{ selectedPost.title }}</h2>
              <p class="muted">{{ selectedPost.excerpt }}</p>
              <div class="post-meta">
                <span>Por {{ selectedPost.author || 'Equipe Keller' }}</span>
                <span>{{ formatDate(selectedPost.created_at) }}</span>
              </div>
            </div>
            <img
              v-if="selectedPost.cover_url"
              :src="selectedPost.cover_url"
              alt="Capa do post"
            />
          </div>

          <div class="post-body">
            <component
              v-for="(block, index) in selectedPost.content"
              :is="resolveBlock(block)"
              :key="index"
              :block="block"
            />
          </div>
        </div>

        <div v-else class="empty">
          <h2>Selecione um post</h2>
          <p class="muted">
            Use a coluna à esquerda para explorar conteúdos sobre ofensiva,
            defensiva e resposta a incidentes.
          </p>
        </div>

        <section class="labs" id="labs">
          <h3>Laboratório Keller</h3>
          <p class="muted">
            Coleções de ferramentas, scripts e ambientes seguros para testar
            vulnerabilidades e simular ataques de forma responsável.
          </p>
          <div class="labs-grid">
            <div class="lab-card">
              <h4>Playbooks</h4>
              <p>Modelos de resposta para incidentes e crise digital.</p>
            </div>
            <div class="lab-card">
              <h4>Threat Feeds</h4>
              <p>Listas curadas de IOC, TTPs e alertas do mercado.</p>
            </div>
            <div class="lab-card">
              <h4>Code Snippets</h4>
              <p>Blocos de código prontos para automação e detecção.</p>
            </div>
          </div>
        </section>
      </section>
    </main>

    <div v-if="showAuth" class="modal-backdrop" @click.self="showAuth = false">
      <div class="modal">
        <div class="modal-header">
          <h3>Admin discreto</h3>
          <button class="ghost" @click="showAuth = false">Fechar</button>
        </div>
        <p class="muted">
          Autentique-se para publicar novos posts e gerenciar o conteúdo.
        </p>
        <form class="form" @submit.prevent="handleLogin">
          <label>
            Email
            <input v-model="auth.email" type="email" required />
          </label>
          <label>
            Senha
            <input v-model="auth.password" type="password" required />
          </label>
          <button class="primary" type="submit" :disabled="authLoading">
            {{ authLoading ? 'Entrando...' : 'Entrar' }}
          </button>
          <p v-if="authError" class="alert">{{ authError }}</p>
        </form>
      </div>
    </div>

    <div v-if="session" class="admin-panel">
      <div class="panel-header">
        <h3>Painel do administrador</h3>
        <div class="panel-actions">
          <button class="ghost" @click="handleLogout">Sair</button>
        </div>
      </div>

      <form class="form" @submit.prevent="handleCreatePost">
        <div class="form-grid">
          <label>
            Título
            <input v-model="draft.title" type="text" required />
          </label>
          <label>
            Categoria
            <input v-model="draft.category" type="text" placeholder="Threat Intel" />
          </label>
        </div>
        <label>
          Resumo
          <textarea v-model="draft.excerpt" rows="3" required></textarea>
        </label>
        <label>
          Imagem de capa (URL)
          <input v-model="draft.cover_url" type="url" placeholder="https://" />
        </label>

        <div class="block-builder">
          <div class="block-header">
            <h4>Blocos de conteúdo</h4>
            <div class="block-actions">
              <select v-model="newBlockType">
                <option value="paragraph">Parágrafo</option>
                <option value="image">Imagem</option>
                <option value="video">Vídeo</option>
                <option value="code">Código</option>
                <option value="embed">Embed</option>
                <option value="quote">Citação</option>
              </select>
              <button class="secondary" type="button" @click="addBlock">
                Adicionar bloco
              </button>
            </div>
          </div>

          <div class="blocks">
            <div v-for="(block, index) in draft.content" :key="index" class="block">
              <div class="block-title">
                <strong>{{ block.type }}</strong>
                <button class="ghost" type="button" @click="removeBlock(index)">
                  Remover
                </button>
              </div>

              <template v-if="block.type === 'paragraph'">
                <textarea v-model="block.text" rows="3"></textarea>
              </template>

              <template v-else-if="block.type === 'image'">
                <input v-model="block.url" type="url" placeholder="https://" />
                <input v-model="block.caption" type="text" placeholder="Legenda" />
              </template>

              <template v-else-if="block.type === 'video'">
                <input v-model="block.url" type="url" placeholder="Link do vídeo" />
              </template>

              <template v-else-if="block.type === 'code'">
                <input v-model="block.language" type="text" placeholder="Linguagem" />
                <textarea v-model="block.code" rows="4"></textarea>
              </template>

              <template v-else-if="block.type === 'embed'">
                <textarea
                  v-model="block.embed"
                  rows="4"
                  placeholder="Cole o código do embed/iframe"
                ></textarea>
              </template>

              <template v-else-if="block.type === 'quote'">
                <textarea v-model="block.text" rows="2"></textarea>
                <input v-model="block.author" type="text" placeholder="Autor" />
              </template>
            </div>
          </div>
        </div>

        <button class="primary" type="submit" :disabled="saving">
          {{ saving ? 'Salvando...' : 'Publicar post' }}
        </button>
        <p v-if="saveError" class="alert">{{ saveError }}</p>
      </form>
    </div>

    <footer class="footer">
      <p>© 2024 Keller Solutions — Cybersecurity Intelligence</p>
    </footer>
  </div>
</template>

<script setup>
import { computed, onMounted, ref } from 'vue'
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY
const supabase = supabaseUrl && supabaseKey ? createClient(supabaseUrl, supabaseKey) : null

const loading = ref(true)
const error = ref('')
const posts = ref([])
const selectedPost = ref(null)

const showAuth = ref(false)
const auth = ref({ email: '', password: '' })
const authLoading = ref(false)
const authError = ref('')
const session = ref(null)

const saving = ref(false)
const saveError = ref('')
const newBlockType = ref('paragraph')

const draft = ref({
  title: '',
  excerpt: '',
  cover_url: '',
  category: '',
  content: [],
})

const fallbackPosts = [
  {
    id: 1,
    title: 'Mapa de ameaças: tendências 2024',
    excerpt:
      'Análise dos vetores mais críticos, incluindo ransomware-as-a-service e ataques à cadeia de suprimentos.',
    cover_url:
      'https://images.unsplash.com/photo-1510511459019-5dda7724fd87?auto=format&fit=crop&w=1200&q=80',
    created_at: new Date().toISOString(),
    category: 'Threat Intel',
    author: 'Equipe Keller',
    content: [
      {
        type: 'paragraph',
        text:
          'O cenário de ameaças continua acelerado. Neste relatório reunimos indicadores, campanhas ativas e recomendações imediatas.',
      },
      {
        type: 'image',
        url: 'https://images.unsplash.com/photo-1545239351-1141bd82e8a6?auto=format&fit=crop&w=1200&q=80',
        caption: 'Mapa de incidentes monitorados em 90 dias.',
      },
      {
        type: 'code',
        language: 'bash',
        code: 'grep -R "IOC" /var/log/security | sort -u',
      },
      {
        type: 'video',
        url: 'https://www.youtube.com/embed/aqz-KE-bpKQ',
      },
    ],
  },
  {
    id: 2,
    title: 'Checklist de hardening para ambientes cloud',
    excerpt:
      'Lista prática de controles, automações e regras para reduzir a superfície de ataque no cloud.',
    cover_url:
      'https://images.unsplash.com/photo-1487058792275-0ad4aaf24ca7?auto=format&fit=crop&w=1200&q=80',
    created_at: new Date(Date.now() - 86400000).toISOString(),
    category: 'Defensive',
    author: 'Equipe Keller',
    content: [
      {
        type: 'paragraph',
        text:
          'Checklist dividido por identidade, rede, storage e observabilidade para apoiar operações de segurança.',
      },
      {
        type: 'embed',
        embed:
          '<iframe src="https://www.supabase.com" title="Supabase" loading="lazy"></iframe>',
      },
      {
        type: 'quote',
        text: 'Automação consistente é a base de uma boa postura de segurança.',
        author: 'Keller Cyber',
      },
    ],
  },
]

const resolveBlock = (block) => {
  switch (block.type) {
    case 'image':
      return ImageBlock
    case 'video':
      return VideoBlock
    case 'code':
      return CodeBlock
    case 'embed':
      return EmbedBlock
    case 'quote':
      return QuoteBlock
    default:
      return ParagraphBlock
  }
}

const fetchPosts = async () => {
  loading.value = true
  error.value = ''

  if (!supabase) {
    posts.value = fallbackPosts
    selectedPost.value = fallbackPosts[0]
    loading.value = false
    return
  }

  const { data, error: fetchError } = await supabase
    .from('posts')
    .select('*')
    .eq('published', true)
    .order('created_at', { ascending: false })

  if (fetchError) {
    error.value = fetchError.message
    posts.value = fallbackPosts
    selectedPost.value = fallbackPosts[0]
  } else {
    posts.value = data || []
    selectedPost.value = data?.[0] || null
  }

  loading.value = false
}

const selectPost = (post) => {
  selectedPost.value = post
}

const formatDate = (value) => {
  if (!value) return ''
  return new Date(value).toLocaleDateString('pt-BR', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

const handleLogin = async () => {
  authError.value = ''
  if (!supabase) {
    authError.value = 'Configure o Supabase para autenticar.'
    return
  }

  authLoading.value = true
  const { data, error: signInError } = await supabase.auth.signInWithPassword({
    email: auth.value.email,
    password: auth.value.password,
  })

  if (signInError) {
    authError.value = signInError.message
  } else {
    session.value = data.session
    showAuth.value = false
  }

  authLoading.value = false
}

const handleLogout = async () => {
  if (!supabase) return
  await supabase.auth.signOut()
  session.value = null
}

const slugify = (value) =>
  value
    .toLowerCase()
    .normalize('NFD')
    .replace(/\p{Diacritic}/gu, '')
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/(^-|-$)+/g, '')

const addBlock = () => {
  const type = newBlockType.value
  const base = { type }

  if (type === 'paragraph') base.text = ''
  if (type === 'image') {
    base.url = ''
    base.caption = ''
  }
  if (type === 'video') base.url = ''
  if (type === 'code') {
    base.language = ''
    base.code = ''
  }
  if (type === 'embed') base.embed = ''
  if (type === 'quote') {
    base.text = ''
    base.author = ''
  }

  draft.value.content.push(base)
}

const removeBlock = (index) => {
  draft.value.content.splice(index, 1)
}

const handleCreatePost = async () => {
  saveError.value = ''
  if (!supabase) {
    saveError.value = 'Configure o Supabase para publicar.'
    return
  }

  saving.value = true
  const payload = {
    title: draft.value.title,
    excerpt: draft.value.excerpt,
    cover_url: draft.value.cover_url,
    category: draft.value.category,
    slug: slugify(draft.value.title),
    content: draft.value.content,
    published: true,
  }

  const { error: insertError } = await supabase.from('posts').insert(payload)

  if (insertError) {
    saveError.value = insertError.message
  } else {
    draft.value = { title: '', excerpt: '', cover_url: '', category: '', content: [] }
    await fetchPosts()
  }

  saving.value = false
}

onMounted(async () => {
  if (supabase) {
    const { data } = await supabase.auth.getSession()
    session.value = data.session
  }
  await fetchPosts()
})

const ParagraphBlock = (props) => ({
  props: ['block'],
  template: '<p class="block-paragraph">{{ block.text }}</p>',
})

const ImageBlock = (props) => ({
  props: ['block'],
  template: `
    <figure class="block-media">
      <img :src="block.url" alt="Imagem do post" />
      <figcaption v-if="block.caption">{{ block.caption }}</figcaption>
    </figure>
  `,
})

const VideoBlock = (props) => ({
  props: ['block'],
  template: `
    <div class="block-media video">
      <iframe :src="block.url" title="Vídeo" frameborder="0" allowfullscreen></iframe>
    </div>
  `,
})

const CodeBlock = (props) => ({
  props: ['block'],
  template: `
    <pre class="block-code"><code>{{ block.code }}</code></pre>
  `,
})

const EmbedBlock = (props) => ({
  props: ['block'],
  template: `
    <div class="block-media embed" v-html="block.embed"></div>
  `,
})

const QuoteBlock = (props) => ({
  props: ['block'],
  template: `
    <blockquote class="block-quote">
      <p>"{{ block.text }}"</p>
      <cite v-if="block.author">— {{ block.author }}</cite>
    </blockquote>
  `,
})
</script>
