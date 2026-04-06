# Seguranca-CORS-Java

> **Grupo:** Intranet  
> **Data:** 06/04/2026

---

Configuração de CORS em Projetos Java## Introdução

Ao desenvolver aplicações Java, especialmente APIs, é comum precisar lidar com **CORS (Cross-Origin Resource Sharing)**. Essa configuração permite que um frontend (em outro domínio, porta ou protocolo) consiga acessar os recursos do backend.

Uma dúvida frequente é se essa configuração precisa ser aplicada em cada classe ou endpoint — e a resposta é: **não**.

## Configuração Recomendada

A melhor prática é **centralizar a configuração de CORS**, aplicando-a globalmente na aplicação. Isso evita repetição de código e facilita a manutenção.

## Opção 1: Configuração Global com `CorsFilter`

Essa abordagem aplica as regras de CORS para toda a aplicação:

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.*;
import org.springframework.web.filter.CorsFilter;

@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowCredentials(true);
        config.addAllowedOrigin("*"); // Substituir pelo domínio do frontend em produção
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        return new CorsFilter(source);
    }
}
## Opção 2: Configuração com `WebMvcConfigurer`

Outra forma organizada de configurar CORS globalmente:

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.*;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("*")
                .allowedHeaders("*");
    }
}
## Opção 3: Configuração por Endpoint

Caso seja necessário aplicar regras específicas para determinados endpoints, é possível usar a anotação `@CrossOrigin`:

@CrossOrigin(origins = "*")
@RestController
public class MeuController {
}
**Observação:** Essa abordagem não é recomendada como padrão, pois exige repetição em múltiplas classes.

## Boas Práticas

- Evitar o uso de `"*"` em ambiente de produção; prefira especificar o domínio do frontend.

- Priorizar configurações globais para manter o código mais limpo e organizado.

- Utilizar configurações específicas por endpoint apenas quando houver necessidade real.

## Conclusão

Não é necessário configurar CORS em cada arquivo da aplicação. A melhor abordagem é centralizar essa configuração em uma única classe, garantindo consistência, segurança e facilidade de manutenção.