<template>
  <div class="movie-app">
    <header class="app-header">
      <h1>🎬 Movie Search TMDB</h1>
      <p>Encontre informações sobre seus filmes favoritos</p>
    </header>

    <main class="main-content">
      <!-- Barra de pesquisa principal -->
      <div class="search-container">
        <!-- Campo de pesquisa por título -->
        <input 
          v-model="searchQuery" 
          placeholder="Digite o nome do filme..." 
          @keyup.enter="handleSearch"
          class="search-input"
        />

        <!-- Campo para filtro por ano -->
        <input 
          v-model.number="filterYear" 
          placeholder="Ano (ex: 2023)" 
          type="number" 
          min="1800"
          max="2025"
          class="year-input"
        />

        <!-- Botão de pesquisa geral (título + ano se ambos preenchidos) -->
        <button @click="handleSearch" :disabled="loading" class="search-btn">
          {{ loading ? '🔍 Buscando...' : '🔍 Pesquisar' }}
        </button>
        
        <!-- Botão para pesquisar apenas por ano -->
        <button 
          @click="searchByYearOnly" 
          :disabled="loading || !filterYear" 
          class="year-search-btn"
        >
          🗓️ Buscar por Ano
        </button>
        
        <!-- Botão para carregar filmes populares -->
        <button @click="getPopularMovies" :disabled="loading" class="popular-btn">
          🎉 Filmes Populares
        </button>
        
        <!-- Botão para limpar pesquisa / resultados -->
        <button v-if="showClearButton" @click="clearSearch" class="clear-btn">
          🗑️ Limpar
        </button>
      </div>

      <!-- Informações da busca atual (tipo de pesquisa) -->
      <div v-if="currentSearchType" class="search-info">
        <p>
          <span v-if="currentSearchType === 'title'">
            🔍 Pesquisando por título: "{{ searchQuery }}"
          </span>
          <span v-if="currentSearchType === 'year'">
            🗓️ Mostrando filmes do ano: {{ filterYear }}
          </span>
          <span v-if="currentSearchType === 'both'">
            🎯 Pesquisando "{{ searchQuery }}" do ano {{ filterYear }}
          </span>
          <span v-if="currentSearchType === 'popular'">
            🎉 Filmes Mais Populares
          </span>
        </p>
      </div>

      <!-- Mensagens de status de carregamento / erro -->
      <div v-if="loading" class="status-message loading">
        <div class="spinner"></div>
        <p>Carregando filmes...</p>
      </div>

      <div v-if="error" class="status-message error">
        <p>❌ {{ error }}</p>
      </div>

      <!-- Lista de filmes quando há resultados -->
      <div v-if="movies.length > 0" class="movies-section">
        <h2>
          <span v-if="currentSearchType === 'title'">
            🎭 Resultados para "{{ searchQuery }}"
          </span>
          <span v-if="currentSearchType === 'year'">
            🗓️ Filmes de {{ filterYear }}
          </span>
          <span v-if="currentSearchType === 'both'">
            🎯 "{{ searchQuery }}" em {{ filterYear }}
          </span>
          <span v-if="currentSearchType === 'popular'">
            🎉 Filmes Mais Populares
          </span>
          <span class="results-count">
            ({{ filteredMovies.length }} filmes)
          </span>
        </h2>
        
        <div class="movies-grid">
          <!-- Card de cada filme -->
          <div 
            v-for="movie in filteredMovies" 
            :key="movie.id" 
            @click="getMovieDetails(movie.id)"
            class="movie-card"
          >
            <div class="movie-poster">
              <img 
                :src="getPosterUrl(movie.poster_path)" 
                :alt="movie.title"
                @error="handleImageError"
              />
              <!-- Overlay ao passar o rato no poster -->
              <div class="movie-overlay">
                <span>Ver Detalhes</span>
              </div>

              <!-- Posição no ranking quando for lista de populares -->
              <div class="movie-rank" v-if="currentSearchType === 'popular'">
                #{{ movies.indexOf(movie) + 1 }}
              </div>
            </div>

            <!-- Informações principais do filme no card -->
            <div class="movie-info">
              <h3 class="movie-title">{{ movie.title }}</h3>
              <p class="movie-year">{{ formatYear(movie.release_date) }}</p>
              <p class="movie-rating">
                ⭐ {{ movie.vote_average?.toFixed(1) }}/10
              </p>
              <p class="movie-votes">
                👥 {{ movie.vote_count }} votos
              </p>
            </div>
          </div>
        </div>
      </div>

      <!-- Mensagem quando não há resultados para a pesquisa atual -->
      <div 
        v-else-if="(searchQuery || filterYear) && !loading" 
        class="status-message no-results"
      >
        <p v-if="currentSearchType === 'title'">
          Nenhum filme encontrado para "{{ searchQuery }}"
        </p>
        <p v-if="currentSearchType === 'year'">
          Nenhum filme encontrado para o ano {{ filterYear }}
        </p>
        <p v-if="currentSearchType === 'both'">
          Nenhum filme encontrado para "{{ searchQuery }}" no ano {{ filterYear }}
        </p>
      </div>

      <!-- Secção inicial mostrando filmes populares (quando não há pesquisa) -->
      <div 
        v-else-if="!searchQuery && !filterYear && !loading && popularMovies.length > 0" 
        class="initial-section"
      >
        <h2>🎉 Filmes Mais Populares</h2>
        <p class="subtitle">Descubra os filmes mais populares do momento</p>
        
        <div class="movies-grid">
          <div 
            v-for="movie in popularMovies" 
            :key="movie.id" 
            @click="getMovieDetails(movie.id)"
            class="movie-card"
          >
            <div class="movie-poster">
              <img 
                :src="getPosterUrl(movie.poster_path)" 
                :alt="movie.title"
                @error="handleImageError"
              />
              <div class="movie-overlay">
                <span>Ver Detalhes</span>
              </div>
              <div class="movie-rank">
                #{{ popularMovies.indexOf(movie) + 1 }}
              </div>
            </div>
            <div class="movie-info">
              <h3 class="movie-title">{{ movie.title }}</h3>
              <p class="movie-year">{{ formatYear(movie.release_date) }}</p>
              <p class="movie-rating">
                ⭐ {{ movie.vote_average?.toFixed(1) }}/10
              </p>
              <p class="movie-votes">
                👥 {{ movie.vote_count }} votos
              </p>
            </div>
          </div>
        </div>
      </div>

      <!-- Mensagem de boas-vindas quando ainda não há nada carregado -->
      <div v-else-if="!loading" class="status-message welcome-message">
        <div class="welcome-content">
          <h3>🎬 Bem-vindo ao Movie Search!</h3>
          <p>Encontre informações sobre seus filmes favoritos</p>
          <div class="features-grid">
            <div class="feature-card">
              <div class="feature-icon">🔍</div>
              <h4>Pesquisa por Título</h4>
              <p>Encontre filmes pelo nome</p>
            </div>
            <div class="feature-card">
              <div class="feature-icon">🗓️</div>
              <h4>Pesquisa por Ano</h4>
              <p>Descubra filmes por ano de lançamento</p>
            </div>
            <div class="feature-card">
              <div class="feature-icon">🎉</div>
              <h4>Filmes Populares</h4>
              <p>Veja os filmes mais populares</p>
            </div>
            <div class="feature-card">
              <div class="feature-icon">📋</div>
              <h4>Detalhes Completos</h4>
              <p>Informações detalhadas sobre cada filme</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Overlay de detalhes do filme selecionado -->
      <div v-if="selectedMovie" class="movie-details-overlay">
        <div class="movie-details">
          <!-- Botão para fechar detalhes -->
          <button @click="selectedMovie = null" class="close-btn">
            ✕ Fechar
          </button>
          
          <div class="details-content">
            <!-- Poster em tamanho maior -->
            <div class="details-poster">
              <img 
                :src="getPosterUrl(selectedMovie.poster_path, 'large')" 
                :alt="selectedMovie.title"
              />
            </div>
            
            <!-- Informações detalhadas do filme -->
            <div class="details-info">
              <h2>{{ selectedMovie.title }}</h2>
              
              <!-- Metadados principais: ano, rating, votos, duração -->
              <div class="movie-meta">
                <span class="year">
                  📅 {{ formatYear(selectedMovie.release_date) }}
                </span>
                <span class="rating">
                  ⭐ {{ selectedMovie.vote_average?.toFixed(1) }}/10
                </span>
                <span class="votes">
                  👥 {{ selectedMovie.vote_count }} votos
                </span>
                <span class="runtime" v-if="selectedMovie.runtime">
                  ⏱️ {{ selectedMovie.runtime }}min
                </span>
              </div>

              <!-- Géneros do filme -->
              <div class="genres" v-if="selectedMovie.genres">
                <span 
                  v-for="genre in selectedMovie.genres" 
                  :key="genre.id" 
                  class="genre-tag"
                >
                  {{ genre.name }}
                </span>
              </div>

              <!-- Sinopse -->
              <p class="overview">
                {{ selectedMovie.overview || 'Sinopse não disponível.' }}
              </p>

              <!-- Grid com mais detalhes (produção, idiomas, orçamento, receita) -->
              <div class="details-grid">
                <div 
                  class="detail-item" 
                  v-if="selectedMovie.production_companies?.length"
                >
                  <strong>Produção:</strong> 
                  {{ selectedMovie.production_companies.map(company => company.name).join(', ') }}
                </div>
                
                <div 
                  class="detail-item" 
                  v-if="selectedMovie.spoken_languages?.length"
                >
                  <strong>Idiomas:</strong> 
                  {{ selectedMovie.spoken_languages.map(lang => lang.english_name).join(', ') }}
                </div>
                
                <div class="detail-item" v-if="selectedMovie.budget">
                  <strong>Orçamento:</strong> 
                  {{ formatCurrency(selectedMovie.budget) }}
                </div>
                
                <div class="detail-item" v-if="selectedMovie.revenue">
                  <strong>Receita:</strong> 
                  {{ formatCurrency(selectedMovie.revenue) }}
                </div>
              </div>

              <!-- Lista de elenco principal (primeiros 6 actores) -->
              <div class="cast-section" v-if="selectedMovie.credits?.cast">
                <h4>🎭 Elenco Principal</h4>
                <div class="cast-list">
                  <span 
                    v-for="actor in selectedMovie.credits.cast.slice(0, 6)" 
                    :key="actor.id"
                    class="cast-member"
                  >
                    {{ actor.name }}
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
// CONFIGURAÇÃO DA API TMDB
// Chave da API (idealmente deveria vir de variável de ambiente em produção)
const API_KEY = 'a07e325eca9bdfabf797ac882c32ded0';
// URL base para endpoints
const BASE_URL = 'https://api.themoviedb.org/3';
// URL base para imagens de posters
const IMAGE_BASE_URL = 'https://image.tmdb.org/t/p';

export default {
  name: 'MovieSearch',

  data() {
    return {
      // Texto digitado pelo utilizador para pesquisa por título
      searchQuery: '',
      // Ano usado como filtro (pode vir do input de número)
      filterYear: '',
      // Lista de filmes retornados pela pesquisa corrente
      movies: [],
      // Lista de filmes populares (usada na página inicial)
      popularMovies: [],
      // Objeto com detalhes do filme selecionado (para o modal)
      selectedMovie: null,
      // Estado de carregamento (true enquanto a API está a responder)
      loading: false,
      // Mensagem de erro (quando algo corre mal)
      error: '',
      // Tipo de pesquisa atual: 'title', 'year', 'both', 'popular' ou ''
      currentSearchType: ''
    }
  },

  computed: {
    /**
     * filteredMovies:
     * - Se a pesquisa é apenas por ano, devolve diretamente this.movies.
     * - Caso contrário, aplica o filtro de ano (se definido) em cima da lista de filmes.
     */
    filteredMovies() {
      if (this.currentSearchType === 'year') {
        // Pesquisa por ano já retorna os filmes daquele ano
        return this.movies;
      }
      
      // Se não há filtro de ano, devolve a lista completa
      if (!this.filterYear) return this.movies;
      
      // Filtra pelo ano a partir da data de lançamento
      return this.movies.filter(movie => {
        const movieYear = new Date(movie.release_date).getFullYear();
        return movieYear == this.filterYear;
      });
    },
    
    /**
     * showClearButton:
     * - Controla se o botão "Limpar" deve ser mostrado.
     * - É mostrado quando há filmes, texto de pesquisa ou filtro de ano.
     */
    showClearButton() {
      return this.movies.length > 0 || this.searchQuery || this.filterYear;
    }
  },

  async mounted() {
    // Quando o componente é montado, carrega de imediato os filmes populares
    await this.getPopularMovies();
  },

  methods: {
    /**
     * handleSearch:
     * - Decide qual URL usar com base na combinação de título e ano.
     * - Faz chamada à API TMDB para pesquisar filmes.
     */
    async handleSearch() {
      // Caso não tenha nem título nem ano, mostra uma mensagem de alerta
      if (!this.searchQuery.trim() && !this.filterYear) {
        this.error = 'Por favor, digite um título ou ano para pesquisar.';
        return;
      }

      this.loading = true;
      this.error = '';
      this.movies = [];
      this.selectedMovie = null;

      try {
        let url;
        
        if (this.searchQuery && this.filterYear) {
          // Pesquisa por título E ano
          this.currentSearchType = 'both';
          url = `${BASE_URL}/search/movie?api_key=${API_KEY}&query=${encodeURIComponent(this.searchQuery)}&year=${this.filterYear}&language=pt-BR&page=1`;
        } else if (this.searchQuery) {
          // Pesquisa apenas por título
          this.currentSearchType = 'title';
          url = `${BASE_URL}/search/movie?api_key=${API_KEY}&query=${encodeURIComponent(this.searchQuery)}&language=pt-BR&page=1`;
        } else {
          // Pesquisa apenas por ano usando o endpoint "discover"
          this.currentSearchType = 'year';
          url = `${BASE_URL}/discover/movie?api_key=${API_KEY}&primary_release_year=${this.filterYear}&sort_by=popularity.desc&language=pt-BR&page=1`;
        }

        const response = await fetch(url);
        const data = await response.json();

        // Se houver resultados, guarda na lista de filmes;
        // caso contrário, mostra mensagem adequada
        if (data.results && data.results.length > 0) {
          this.movies = data.results;
        } else {
          this.setErrorMessage();
        }

      } catch (err) {
        // Erro genérico de conexão ou API
        this.error = 'Erro ao conectar com a API. Tente novamente.';
        console.error('Erro TMDB:', err);
      } finally {
        this.loading = false;
      }
    },

    /**
     * searchByYearOnly:
     * - Força a pesquisa apenas pelo ano, limpando o título.
     */
    async searchByYearOnly() {
      if (!this.filterYear) {
        this.error = 'Por favor, digite um ano para pesquisar.';
        return;
      }

      // Limpa o título para deixar claro que a pesquisa é só por ano
      this.searchQuery = '';
      this.handleSearch();
    },

    /**
     * getPopularMovies:
     * - Carrega a lista de filmes populares da API TMDB.
     * - Usado tanto na montagem inicial como quando o utilizador clica em "Filmes Populares".
     */
    async getPopularMovies() {
      this.loading = true;
      this.error = '';
      this.currentSearchType = 'popular';
      this.searchQuery = '';
      this.filterYear = '';

      try {
        const response = await fetch(
          `${BASE_URL}/movie/popular?api_key=${API_KEY}&language=pt-BR&page=1`
        );
        
        const data = await response.json();

        if (data.results && data.results.length > 0) {
          this.popularMovies = data.results;
          // Limpa resultados de pesquisas anteriores
          this.movies = [];
        } else {
          this.error = 'Erro ao carregar filmes populares.';
        }

      } catch (err) {
        this.error = 'Erro ao carregar filmes populares. Tente novamente.';
        console.error('Erro TMDB:', err);
      } finally {
        this.loading = false;
      }
    },

    /**
     * setErrorMessage:
     * - Define uma mensagem de erro específica baseada no tipo de pesquisa atual.
     */
    setErrorMessage() {
      switch (this.currentSearchType) {
        case 'title':
          this.error = `Nenhum filme encontrado para "${this.searchQuery}"`;
          break;
        case 'year':
          this.error = `Nenhum filme encontrado para o ano ${this.filterYear}`;
          break;
        case 'both':
          this.error = `Nenhum filme encontrado para "${this.searchQuery}" no ano ${this.filterYear}`;
          break;
        default:
          this.error = 'Nenhum filme encontrado';
      }
    },

    /**
     * getMovieDetails:
     * - Carrega detalhes de um filme específico (incluindo créditos/elenco).
     * - Abre o modal de detalhes ao preencher selectedMovie.
     */
    async getMovieDetails(movieId) {
      this.loading = true;
      this.error = '';

      try {
        const response = await fetch(
          `${BASE_URL}/movie/${movieId}?api_key=${API_KEY}&append_to_response=credits&language=pt-BR`
        );
        
        const data = await response.json();

        if (data) {
          // Guarda todos os detalhes do filme selecionado
          this.selectedMovie = data;
        } else {
          this.error = 'Erro ao carregar detalhes do filme.';
        }

      } catch (err) {
        this.error = 'Erro ao carregar detalhes. Tente novamente.';
        console.error('Erro TMDB:', err);
      } finally {
        this.loading = false;
      }
    },

    /**
     * getPosterUrl:
     * - Monta a URL da imagem do poster a partir do caminho retornado pela API.
     * - Aceita tamanhos "small", "medium" e "large".
     * - Se não houver poster, devolve uma imagem placeholder em SVG (base64).
     */
    getPosterUrl(posterPath, size = 'medium') {
      const sizes = {
        small: 'w185',
        medium: 'w342',
        large: 'w500'
      };
      
      if (!posterPath) {
        // Placeholder (SVG codificado em base64) para quando não há poster
        return 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMzAwIiBoZWlnaHQ9IjQ1MCIgdmlld0JveD0iMCAwIDMwMCA0NTAiIGZpbGw9Im5vbmUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHJlY3Qgd2lkdGg9IjMwMCIgaGVpZ2h0PSI0NTAiIGZpbGw9IiNGM0Y0RjYiLz48cGF0aCBkPSJNMTIwIDE4MEgxODBWMjQwSDEyMFYxODBaIiBmaWxsPSIjOTlBQUFGIi8+PHBhdGggZD0iTTkwIDEyMEgyMTBWMTgwSDIxMEg5MFYxMjBaIiBmaWxsPSIjOTlBQUFGIi8+PC9zdmc+';
      }
      
      return `${IMAGE_BASE_URL}/${sizes[size]}${posterPath}`;
    },

    /**
     * handleImageError:
     * - Callback quando a imagem do poster falha ao carregar.
     * - Substitui por um placeholder genérico.
     */
    handleImageError(event) {
      event.target.src = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMzAwIiBoZWlnaHQ9IjQ1MCIgdmlld0JveD0iMCAwIDMwMCA0NTAiIGZpbGw9Im5vbmUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHJlY3Qgd2lkdGg9IjMwMCIgaGVpZ2h0PSI0NTAiIGZpbGw9IiNGM0Y0RjYiLz48cGF0aCBkPSJNMTIwIDE4MEgxODBWMjQwSDEyMFYxODBaIiBmaWxsPSIjOTlBQUFGIi8+PHBhdGggZD0iTTkwIDEyMEgyMTBWMTgwSDIxMEg5MFYxMjBaIiBmaWxsPSIjOTlBQUFGIi8+PC9zdmc+';
    },

    /**
     * formatYear:
     * - Extrai apenas o ano da data de lançamento (release_date).
     * - Se não houver data, devolve 'N/A'.
     */
    formatYear(dateString) {
      if (!dateString) return 'N/A';
      return new Date(dateString).getFullYear();
    },

    /**
     * formatCurrency:
     * - Formata um número como moeda em USD (usado para orçamento e receita).
     * - Se não houver valor, devolve 'N/A'.
     */
    formatCurrency(amount) {
      if (!amount) return 'N/A';
      return new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD'
      }).format(amount);
    },

    /**
     * clearSearch:
     * - Limpa os campos de pesquisa, lista de filmes e mensagens de erro.
     * - Não apaga a lista de filmes populares (continua disponível).
     */
    clearSearch() {
      this.searchQuery = '';
      this.filterYear = '';
      this.movies = [];
      this.selectedMovie = null;
      this.error = '';
      this.currentSearchType = '';
    }
  }
}
</script>

<style scoped>
/* Reset básico */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* Fonte base e fundo padrão */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f5f5f5;
}

/* Container principal da app */
.movie-app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

/* Cabeçalho da aplicação */
.app-header {
  background: rgba(0, 0, 0, 0.8);
  color: white;
  text-align: center;
  padding: 2rem;
  backdrop-filter: blur(10px);
}

.app-header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
}

.app-header p {
  opacity: 0.8;
  font-size: 1.1rem;
}

/* Conteúdo principal com largura máxima */
.main-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

/* Container de pesquisa (inputs e botões) */
.search-container {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
}

/* Input de texto principal (título) */
.search-input {
  flex: 1;
  min-width: 300px;
  padding: 12px 16px;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

/* Input de ano */
.year-input {
  width: 120px;
  padding: 12px;
  border: 2px solid #e1e5e9;
  border-radius: 8px;
  font-size: 1rem;
}

/* Botões principais de ação */
.search-btn,
.clear-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

/* Botão de pesquisa */
.search-btn {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

.search-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
}

.search-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Botão de limpar */
.clear-btn {
  background: #dc3545;
  color: white;
}

.clear-btn:hover {
  background: #c82333;
  transform: translateY(-2px);
}

/* Botão para pesquisa por ano apenas */
.year-search-btn {
  padding: 12px 20px;
  background: linear-gradient(135deg, #28a745, #20c997);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.year-search-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
}

.year-search-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: #6c757d;
}

/* Botão de filmes populares */
.popular-btn {
  padding: 12px 20px;
  background: linear-gradient(135deg, #ff6b6b, #ee5a52);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.popular-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
}

.popular-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Caixa com informação do tipo de pesquisa atual */
.search-info {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  text-align: center;
  backdrop-filter: blur(10px);
}

/* Mensagens de status genéricas */
.status-message {
  text-align: center;
  padding: 2rem;
  border-radius: 8px;
  margin: 1rem 0;
}

/* Estado de carregamento */
.loading {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  backdrop-filter: blur(10px);
}

/* Estado de erro */
.error {
  background: rgba(220, 53, 69, 0.1);
  color: white;
  border: 1px solid #dc3545;
}

/* Quando não há resultados */
.no-results {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  backdrop-filter: blur(10px);
}

/* Mensagem de boas-vindas */
.welcome-message {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  backdrop-filter: blur(10px);
  text-align: center;
}

/* Spinner de carregamento */
.spinner {
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top: 4px solid white;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Secção inicial (filmes populares) */
.initial-section {
  margin-top: 2rem;
}

.initial-section h2 {
  color: white;
  text-align: center;
  margin-bottom: 0.5rem;
  font-size: 2rem;
}

.subtitle {
  color: rgba(255, 255, 255, 0.8);
  text-align: center;
  margin-bottom: 2rem;
  font-size: 1.1rem;
}

/* Título da secção de filmes */
.movies-section h2 {
  color: white;
  text-align: center;
  margin-bottom: 2rem;
  font-size: 1.8rem;
}

.results-count {
  font-size: 0.9rem;
  opacity: 0.8;
  font-weight: normal;
}

/* Grid de filmes */
.movies-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 2rem;
  padding: 1rem 0;
}

/* Card individual do filme */
.movie-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
  cursor: pointer;
  transition: all 0.3s ease;
}

.movie-card:hover {
  transform: translateY(-10px);
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.2);
}

/* Área do poster no card */
.movie-poster {
  position: relative;
  overflow: hidden;
  height: 300px;
}

.movie-poster img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s ease;
}

.movie-card:hover .movie-poster img {
  transform: scale(1.1);
}

/* Overlay "Ver Detalhes" */
.movie-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.movie-card:hover .movie-overlay {
  opacity: 1;
}

.movie-overlay span {
  color: white;
  font-weight: bold;
  font-size: 1.1rem;
}

/* Badge do ranking (posição na lista) */
.movie-rank {
  position: absolute;
  top: 10px;
  left: 10px;
  background: linear-gradient(135deg, #ff6b6b, #ee5a52);
  color: white;
  padding: 5px 10px;
  border-radius: 20px;
  font-weight: bold;
  font-size: 0.8rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

/* Texto dentro do card do filme */
.movie-info {
  padding: 1rem;
}

.movie-title {
  font-size: 1.1rem;
  font-weight: bold;
  margin-bottom: 0.5rem;
  color: #333;
  line-height: 1.3;
}

.movie-year {
  color: #667eea;
  font-weight: bold;
  margin-bottom: 0.25rem;
}

.movie-rating {
  color: #ffc107;
  font-weight: bold;
  margin-bottom: 0.25rem;
}

.movie-votes {
  color: #6c757d;
  font-size: 0.8rem;
  margin-top: 0.25rem;
}

/* Overlay de detalhes do filme (modal fullscreen) */
.movie-details-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

/* Caixa de detalhes */
.movie-details {
  background: white;
  border-radius: 15px;
  max-width: 900px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
}

/* Botão para fechar o modal */
.close-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  background: #dc3545;
  color: white;
  border: none;
  padding: 10px 15px;
  border-radius: 50%;
  cursor: pointer;
  z-index: 1001;
  font-size: 1.2rem;
}

/* Layout interno dos detalhes (poster + texto) */
.details-content {
  display: grid;
  grid-template-columns: 300px 1fr;
  gap: 2rem;
  padding: 2rem;
}

.details-poster img {
  width: 100%;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

/* Título do filme no modal */
.details-info h2 {
  font-size: 2rem;
  margin-bottom: 1rem;
  color: #333;
}

/* Metadados (ano, rating, votos, duração) */
.movie-meta {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.movie-meta span {
  background: #f8f9fa;
  padding: 5px 12px;
  border-radius: 20px;
  font-size: 0.9rem;
  color: #666;
}

/* Géneros */
.genres {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.genre-tag {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  padding: 5px 12px;
  border-radius: 15px;
  font-size: 0.8rem;
  font-weight: bold;
}

/* Sinopse */
.overview {
  font-size: 1.1rem;
  line-height: 1.6;
  margin-bottom: 1.5rem;
  color: #555;
}

/* Grid de detalhes adicionais (produção, idiomas, etc.) */
.details-grid {
  display: grid;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.detail-item {
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #667eea;
}

/* Secção do elenco */
.cast-section h4 {
  margin-bottom: 1rem;
  color: #333;
}

.cast-list {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.cast-member {
  background: #e9ecef;
  padding: 5px 10px;
  border-radius: 15px;
  font-size: 0.9rem;
  color: #495057;
}

/* Conteúdo da mensagem de boas-vindas */
.welcome-content {
  max-width: 800px;
  margin: 0 auto;
}

.welcome-content h3 {
  font-size: 2rem;
  margin-bottom: 1rem;
}

.welcome-content p {
  font-size: 1.2rem;
  margin-bottom: 2rem;
  opacity: 0.9;
}

/* Grid com features da app (boas-vindas) */
.features-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-top: 2rem;
}

.feature-card {
  background: rgba(255, 255, 255, 0.1);
  padding: 1.5rem;
  border-radius: 12px;
  text-align: center;
  transition: transform 0.3s ease;
}

.feature-card:hover {
  transform: translateY(-5px);
  background: rgba(255, 255, 255, 0.15);
}

.feature-icon {
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

.feature-card h4 {
  font-size: 1.2rem;
  margin-bottom: 0.5rem;
  color: white;
}

.feature-card p {
  font-size: 0.9rem;
  opacity: 0.8;
  margin: 0;
}

/* Ajustes responsivos para tablets */
@media (max-width: 768px) {
  .search-container {
    flex-direction: column;
  }
  
  .search-input,
  .year-input {
    min-width: auto;
    width: 100%;
  }
  
  .search-btn,
  .year-search-btn,
  .popular-btn,
  .clear-btn {
    width: 100%;
  }
  
  .movies-grid {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  }
  
  .movie-meta {
    justify-content: center;
  }
  
  .genres {
    justify-content: center;
  }
  
  .details-content {
    grid-template-columns: 1fr;
    text-align: center;
  }
  
  .features-grid {
    grid-template-columns: 1fr;
  }
  
  .feature-card {
    padding: 1rem;
  }
  
  .app-header h1 {
    font-size: 2rem;
  }
  
  .main-content {
    padding: 1rem;
  }
}

/* Ajustes responsivos para telemóveis pequenos */
@media (max-width: 480px) {
  .movies-grid {
    grid-template-columns: 1fr;
  }
  
  .movie-poster {
    height: 250px;
  }
  
  .details-content {
    padding: 1rem;
  }
  
  .details-info h2 {
    font-size: 1.5rem;
  }
}
</style>
