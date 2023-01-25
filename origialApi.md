import { rejects } from "assert";
import axios, { AxiosError, AxiosRequestConfig } from "axios";
import { parseCookies, setCookie } from 'nookies';
import { signOut } from "../contexts/AuthContext";

interface AxiosErrorResponse {
  code?: string;
}

let cookies = parseCookies();
let isRefreshing = false;
let failedRequestQueue: any[] = [];

export const api = axios.create({
  baseURL: 'http://localhost:3333',
  headers: {
    Authorization: `Bearer ${cookies['nextauth.token']}`
  }
});

api.interceptors.response.use(response => {
  return response;
}, (error: AxiosError<AxiosErrorResponse>) => {
  if(error.response?.status === 401) {
    if(error.response.data?.code === 'token.expired') {
      cookies = parseCookies();

      const { 'nextauth.refreshToken': refreshToken } = cookies;
      const originalConfig = error.config

     if(!isRefreshing) {
      isRefreshing = true;
      
      api.post('/refresh', {
        refreshToken
      }).then(response => {
        const { token } = response.data;

        setCookie(undefined, 'nextauth.token', token, {
          maxAge: 60 * 60 * 24 * 30, // 30 dias
          path: '/'
        });
        setCookie(undefined, 'nextauth.refreshToken', response.data.refreshToken, {
          maxAge: 60 * 60 * 24 * 30, // 30 dias
          path: '/'
        });

        api.defaults.headers['Authorization'] = `Bearer ${token}`;

        failedRequestQueue.forEach(request => request.onSucess(token));
        failedRequestQueue = [];
      }).catch(err => {
        failedRequestQueue.forEach(request => request.onFailure(err));
        failedRequestQueue = [];

        if(process.browser) {
          signOut()
        }
      }).finally(() => {
        isRefreshing = false;
      });
     }

     return new Promise((resolve, reject) => {
      failedRequestQueue.push({
        onSucess: (token: string) => {
          if(originalConfig?.headers !== undefined) {
            
            originalConfig.headers['Authorization'] = `Bearer ${token}`
            resolve(api(originalConfig))
          }

        },
        onFailure: (err: AxiosError) => {
          reject(err)
        }
      })
     })
    } else {
      if(process.browser) {
        signOut();
      }
    }
  }

  return Promise.reject(error);
})
