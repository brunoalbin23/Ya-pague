-- =============================================
-- CHAUCHA / YA PAGUÉ - Supabase Schema
-- =============================================

-- FAMILIAS
CREATE TABLE familias (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  codigo VARCHAR(6) UNIQUE NOT NULL,
  nombre VARCHAR(100),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- PROFILES
CREATE TABLE profiles (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE NOT NULL,
  nombre VARCHAR(30),
  familia_id UUID REFERENCES familias(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- CATEGORIAS PERSONALIZADAS
CREATE TABLE categorias (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  familia_id UUID REFERENCES familias(id) ON DELETE CASCADE NOT NULL,
  nombre VARCHAR(50) NOT NULL,
  emoji VARCHAR(10),
  color VARCHAR(20),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- GASTOS
CREATE TABLE gastos (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  familia_id UUID REFERENCES familias(id) ON DELETE CASCADE NOT NULL,
  categoria VARCHAR(50) NOT NULL,
  monto NUMERIC(12,2) NOT NULL,
  descripcion TEXT,
  fecha DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- INGRESOS
CREATE TABLE ingresos (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  familia_id UUID REFERENCES familias(id) ON DELETE CASCADE NOT NULL,
  tipo VARCHAR(50) NOT NULL,
  monto NUMERIC(12,2) NOT NULL,
  descripcion TEXT,
  fecha DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================
-- ÍNDICES
-- =============================================
CREATE INDEX idx_gastos_familia_fecha ON gastos(familia_id, fecha);
CREATE INDEX idx_ingresos_familia_fecha ON ingresos(familia_id, fecha);
CREATE INDEX idx_categorias_familia ON categorias(familia_id);
CREATE INDEX idx_profiles_user ON profiles(user_id);
CREATE INDEX idx_profiles_familia ON profiles(familia_id);

-- =============================================
-- TRIGGER: crear perfil automático al registrarse
-- =============================================
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (user_id, nombre)
  VALUES (NEW.id, NEW.raw_user_meta_data->>'nombre');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();

-- =============================================
-- ROW LEVEL SECURITY
-- =============================================

ALTER TABLE familias ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE categorias ENABLE ROW LEVEL SECURITY;
ALTER TABLE gastos ENABLE ROW LEVEL SECURITY;
ALTER TABLE ingresos ENABLE ROW LEVEL SECURITY;

-- FAMILIAS
CREATE POLICY "familia_select" ON familias
  FOR SELECT USING (
    id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "familia_insert" ON familias
  FOR INSERT WITH CHECK (true);
CREATE POLICY "familia_update" ON familias
  FOR UPDATE USING (
    id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );

-- PROFILES
CREATE POLICY "profiles_select" ON profiles
  FOR SELECT USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
    OR user_id = auth.uid()
  );
CREATE POLICY "profiles_insert" ON profiles
  FOR INSERT WITH CHECK (user_id = auth.uid());
CREATE POLICY "profiles_update" ON profiles
  FOR UPDATE USING (user_id = auth.uid());

-- CATEGORIAS
CREATE POLICY "categorias_select" ON categorias
  FOR SELECT USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "categorias_insert" ON categorias
  FOR INSERT WITH CHECK (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "categorias_update" ON categorias
  FOR UPDATE USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "categorias_delete" ON categorias
  FOR DELETE USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );

-- GASTOS
CREATE POLICY "gastos_select" ON gastos
  FOR SELECT USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "gastos_insert" ON gastos
  FOR INSERT WITH CHECK (
    user_id = auth.uid() AND
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "gastos_update" ON gastos
  FOR UPDATE USING (user_id = auth.uid());
CREATE POLICY "gastos_delete" ON gastos
  FOR DELETE USING (user_id = auth.uid());

-- INGRESOS
CREATE POLICY "ingresos_select" ON ingresos
  FOR SELECT USING (
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "ingresos_insert" ON ingresos
  FOR INSERT WITH CHECK (
    user_id = auth.uid() AND
    familia_id IN (SELECT familia_id FROM profiles WHERE user_id = auth.uid())
  );
CREATE POLICY "ingresos_update" ON ingresos
  FOR UPDATE USING (user_id = auth.uid());
CREATE POLICY "ingresos_delete" ON ingresos
  FOR DELETE USING (user_id = auth.uid());