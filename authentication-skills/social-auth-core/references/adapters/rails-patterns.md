# Rails Adapter Patterns

## Purpose

Provides Rails-specific integration patterns for social authentication.

## Relationship to Core Rules

Global security, execution order, stop conditions, and decision hierarchy are defined in:

- ../../../core/SECURITY_INVARIANTS.md
- ../../../core/EXECUTION_RULES.md
- ../../../core/STOP_CONDITIONS.md
- ../../../core/DECISION_MODEL.md

This file defines Rails-specific routing, controller, session, and callback implementation guidance only.

## When to Use

Use this file when social authentication is being implemented in a Ruby on Rails application.

## Route Placement

Define login start and callback routes in config/routes.rb.

Example:

Rails.application.routes.draw do
get "auth/:provider", to: "social_auth#start"
get "auth/:provider/callback", to: "social_auth#callback"
end

## Login Start Pattern

- generate state, nonce, and code_verifier server-side
- derive PKCE challenge using S256
- store pre-auth state in session
- redirect to provider

## Callback Pattern

- validate state, nonce, TTL, and provider
- exchange code for tokens
- validate tokens
- link or create account
- rotate session
- redirect safely

## Minimal Implementation Skeleton

class SocialAuthController < ApplicationController
def start
provider = params[:provider]
state = generate_state
nonce = generate_nonce
verifier = generate_code_verifier
challenge = derive_s256_challenge(verifier)

    session[:oauth_pre_auth] = {
      provider: provider,
      state: state,
      nonce: nonce,
      verifier: verifier
    }

    redirect_to build_authorize_url(provider, state, nonce, challenge), allow_other_host: true

end

def callback
provider = params[:provider]
code = params[:code]
state = params[:state]
pre_auth = session[:oauth_pre_auth]

    assert_valid_pre_auth(pre_auth, provider, state)

    tokens = exchange_code_for_tokens(provider, code, pre_auth[:verifier])
    profile = validate_and_fetch_profile(tokens, pre_auth[:nonce])
    user = link_or_create_account(profile, provider)

    rotate_session_after_login(user)

    redirect_to root_path

end
end
