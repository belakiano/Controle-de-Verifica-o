Sistema de Validação - Corpo de Bombeiros Militares (Relações Públicas)
📖 Sobre o Projeto
Este é um sistema de gerenciamento de fichas e postagens administrativas para corporações, como o Corpo de Bombeiros Militares, SAMU e Relações Públicas. A plataforma permite a visualização pública dos perfis dos integrantes, enquanto o acesso para edição, criação e exclusão de conteúdo é restrito a usuários autenticados com diferentes níveis de permissão.

O projeto é construído como uma aplicação de página única (SPA) utilizando HTML, Tailwind CSS e JavaScript, com todo o backend (banco de dados, autenticação e armazenamento de arquivos) gerenciado pela plataforma Supabase.

✨ Funcionalidades Principais
Visualização Pública: Qualquer visitante pode visualizar a lista de integrantes e seus perfis detalhados, incluindo o histórico de postagens.

Autenticação Segura: Sistema de login por email e senha gerenciado pelo Supabase Auth.

Sistema de Permissões por Cargos:

Admin Chefe / Admin: Controle total para criar, editar e excluir integrantes e postagens de todas as divisões.

Comandante: Permissões de edição restritas aos integrantes da sua própria divisão.

Integrante: Acesso de apenas leitura.

Gerenciamento de Integrantes:

Criação de novas fichas com informações como nome (nick), cargo, divisão, status, TAG e especialização.

Suporte para upload de avatares personalizados, que são armazenados no Supabase Storage.

Postagens Administrativas:

Criação de postagens como promoções, rebaixamentos, medalhas, punições, etc.

Ao registrar uma promoção ou rebaixamento, o sistema automaticamente atualiza o cargo do integrante no perfil dele.

Atualizações em Tempo Real: A interface do site reflete mudanças no banco de dados (novos integrantes, postagens, etc.) instantaneamente, sem a necessidade de recarregar a página.

Design Responsivo: A interface se adapta para uma boa experiência em desktops e dispositivos móveis.

🛠️ Tecnologias Utilizadas
Frontend:

HTML5

Tailwind CSS

JavaScript (ES6+)

Lucide Icons

Backend (BaaS):

Supabase:

Supabase Auth: Para gerenciamento de usuários e login.

Supabase Database: Banco de dados PostgreSQL para armazenar os dados de integrantes e postagens.

Supabase Storage: Para hospedar os avatares dos integrantes e a logo do site.

Row Level Security (RLS): Para garantir que as regras de permissão sejam aplicadas diretamente no banco de dados, tornando o sistema seguro.

🚀 Guia de Instalação e Configuração
Para rodar este projeto, você precisará de uma conta no Supabase.

1. Crie um Projeto no Supabase
Acesse supabase.com, crie uma conta e um novo projeto.

Guarde a URL do Projeto e a chave de API anon public. Você as encontrará em Project Settings > API.

2. Configure o Banco de Dados
No painel do seu projeto Supabase, vá para o SQL Editor e execute os seguintes scripts na ordem:

a) Criação das Tabelas
-- Cria a tabela de membros
CREATE TABLE public.members (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
    name TEXT NOT NULL,
    rank TEXT,
    tag TEXT,
    specialization TEXT,
    status TEXT DEFAULT 'Ativo'::text,
    image_url TEXT,
    join_date TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()),
    division TEXT
);

-- Cria a tabela de postagens
CREATE TABLE public.posts (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    date TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()),
    member_id UUID REFERENCES public.members(id) ON DELETE CASCADE,
    type TEXT,
    content TEXT,
    author TEXT,
    previous_rank TEXT NULL,
    new_rank TEXT NULL
);

-- Cria a tabela de perfis de usuário
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT,
  role TEXT DEFAULT 'integrante',
  division TEXT
);

-- Função para criar um perfil automaticamente para cada novo usuário
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email)
  VALUES (new.id, new.email);
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Gatilho para acionar a função acima
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();

b) Aplicação das Regras de Segurança (RLS)
-- Habilita a segurança em todas as tabelas
ALTER TABLE public.members ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- Regras para 'profiles'
CREATE POLICY "Users can view all profiles" ON public.profiles FOR SELECT USING (true);
CREATE POLICY "Users can update their own profile" ON public.profiles FOR UPDATE USING (auth.uid() = id);

-- Regras para 'members'
CREATE POLICY "Public can view all members" ON public.members FOR SELECT USING (true);
CREATE POLICY "Admins and Commanders can manage their members" ON public.members FOR ALL USING (
  (SELECT role FROM public.profiles WHERE id = auth.uid()) IN ('admin', 'admin_chefe') OR
  ((SELECT role FROM public.profiles WHERE id = auth.uid()) = 'comandante' AND (SELECT division FROM public.profiles WHERE id = auth.uid()) = public.members.division)
);

-- Regras para 'posts'
CREATE POLICY "Public can view all posts" ON public.posts FOR SELECT USING (true);
CREATE POLICY "Admins and Commanders can manage their posts" ON public.posts FOR ALL USING (
  (SELECT role FROM public.profiles WHERE id = auth.uid()) IN ('admin', 'admin_chefe') OR
  ((SELECT role FROM public.profiles WHERE id = auth.uid()) = 'comandante' AND 
   (SELECT division FROM public.profiles WHERE id = auth.uid()) = (SELECT division FROM public.members WHERE id = public.posts.member_id))
);

3. Configure o Armazenamento (Storage)
No painel do Supabase, vá para Storage.

Crie um novo bucket com o nome avatars. Marque-o como público.

Vá para o SQL Editor novamente e execute o seguinte script para aplicar as regras de segurança ao storage:

CREATE POLICY "Allow public read access"
ON storage.objects FOR SELECT
USING ( bucket_id = 'avatars' );

CREATE POLICY "Allow authenticated users to upload"
ON storage.objects FOR INSERT
WITH CHECK ( bucket_id = 'avatars' AND auth.role() = 'authenticated' );

4. Configure o Arquivo index.html
Abra o arquivo index.html e substitua os valores das constantes SUPABASE_URL e SUPABASE_ANON_KEY pelas credenciais que você copiou no primeiro passo.

5. Gerencie os Usuários
Criar Usuários: Vá para a seção Authentication no Supabase para criar novos usuários com email e senha.

Definir Permissões: Vá para o Table Editor e na tabela profiles, edite o role e a division de cada usuário para definir seu nível de acesso.
